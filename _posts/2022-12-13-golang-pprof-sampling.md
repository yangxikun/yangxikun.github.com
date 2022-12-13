---
layout: post
title: "Go PProf 采样的实现"
description: "Go PProf 采样的实现"
category: GoLang
tags: []
---
{% include JB/setup %}

# CPU Profiler

CPU Profiler 能帮助我们分析程序中消耗 CPU Time （包括**用户空间**和**内核空间**的时间）最多的调用栈，我们可以通过如下 API 来获得一份 CPU 消耗的采样结果：

- 通过命令行测试工具：`go test -cpuprofile cpu.pprof`
- 代码中主动开始和停止：`pprof.StartCPUProfile(w)` 和 `pprof.StopCPUProfile()`
- 通过 HTTP 接口：`import _ "net/http/pprof"` 和 `GET /debug/pprof/profile?seconds=30`

虽然提供了 API `runtime.SetCPUProfileRate(hz)` 让我们设置采样的速率，但通常我们并不需要调整。在 `pprof.StartCPUProfile(w)` 中，默认会设置 `runtime.SetCPUProfileRate(100)`，即程序每消耗 10ms CPU Time 就采样一次。

`runtime.SetCPUProfileRate(hz)` 会调用 `setcpuprofilerate(hz)` 设置 CPU 采样的速率为 hz 次每秒，如果 hz <= 0，则会停止 CPU 采样。

在 Go 1.18 之前，`setcpuprofilerate(hz)` 会通过 signal_unix.go 中的 `setProcessCPUProfiler(hz)` 执行系统调用 `setitimer(_ITIMER_PROF, new, old *itimerval)` 设置**进程级别的定时器**，即当所有线程消耗的 CPU Time 达到 1000/hz ms 时，进程会收到 SIGPROF 信号，并且会随机地由任意一个正在运行中的线程（其信号屏蔽字没有 SIGPROF）执行信号处理器。

这会存在一个问题，即当进程消耗大量的 CPU Time 时，会产生大量的 SIGPROF 信号，而 SIGPROF 信号属于**标准信号**，多个连续的标准信号，只能有一个处于 pending 状态，其他会被丢弃，这会导致生成的 profile 数据是不准确的。详情可查看 ISSUE：[runtime/pprof: Linux CPU profiles inaccurate beyond 250% CPU use](https://github.com/golang/go/issues/35057)

所以从 Go 1.18 开始，在 Linux 平台上，`setcpuprofilerate(hz)` 会通过 os_linux.go `setThreadCPUProfiler(hz)` 执行系统调用 `timer_create(_CLOCK_THREAD_CPUTIME_ID, &sevp, &timerid)` 为每个**线程**设置单独的定时器，即当一个线程消耗的 CPU Time 达到 1000/hz ms 时，该线程会定向收到 SIGPROF 信号，然后执行信号处理器。

在 Go 中有一个统一的信号处理器 `sighandler(sig uint32, info *siginfo, ctxt unsafe.Pointer, gp *g)`，如果是 SIGPROF 会调用 `sigprof(pc, sp, lr uintptr, gp *g, mp *m)` 生成当前线程的调用栈，然后调用 `cpuProfile.add(tagPtr *unsafe.Pointer, stk []uintptr)` 记录下来。

> 所以当我们需要压测消耗大量 CPU 的程序，应该使用 Go 1.18 及以上，并且在 Linux 平台下进行。

<!--more-->

# Memory Profiler

Memory Profiler 能帮助我们分析哪些调用栈内存分配比较多、哪些调用栈未释放的内存比较多。它有四种类型的采样数据：

- alloc_objects：分配的对象数量
- alloc_space：分配的字节数
- inuse_objects：未释放的对象数量
- inuse_space：未释放的字节数

我们可以通过如下 API 来获得一份内存分配的采样结果：

- 通过命令行测试工具：`go test -memprofile mem.pprof`
- 代码中获取：`pprof.Lookup("allocs").WriteTo(w, 0)` 或 `pprof.Lookup("heap").WriteTo(w, 0)`
- 通过 HTTP 接口：`import _ "net/http/pprof"` 和 `GET /debug/pprof/allocs?seconds=30`、`GET /debug/pprof/heap?seconds=30`，如果传了 seconds=30 参数，用于获取 30s 内的增量采样数据，否则获取全量采样数据

上述不同方式获取到的采样数据都包括了 alloc_objects/alloc_space/inuse_objects/inuse_space，`go tool pprof` 命令行可以通过 `-sample_index=alloc_objects`，交互式终端可以通过 `(pprof) sample_index=inuse_space` 切换不同类型的采样数据。

获取内存分配采样的不同 API 都是调用：
- pprof.writeHeapInternal(w io.Writer, debug int, defaultSampleType string) error
    - runtime.MemProfile(p []MemProfileRecord, inuseZero bool) (n int, ok bool) 读取已采样的所有内存分配数据
    - pprof.writeHeapProto(w io.Writer, p []runtime.MemProfileRecord, rate int64, defaultSampleType string) error 以 proto 的格式将采样数据写入 w

内存分配的采样点在 malloc.go 中的 mallocgc，由于 mallocgc 是从 mcache.tiny/mcache.alloc/mheap.allocSpan 分配内存，而栈的内存分配是由 stack.go 中的 stackalloc 从 mcache.stackcache/mheap.allocSpan 分配内存，所以采样到的数据并不包含栈内存的分配。

```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	......
	// 内存分配采样
	if rate := MemProfileRate; rate > 0 {
		// c.nextSample 表示下一个采样点
		if rate != 1 && size < c.nextSample {
			c.nextSample -= size
		} else {
			// 当前分配的内存 x 需要采样
			profilealloc(mp, x, size)
		}
	}
	......
}

func profilealloc(mp *m, x unsafe.Pointer, size uintptr) {
   c := getMCache(mp)
   if c == nil {
      throw("profilealloc called without a P or outside bootstrapping")
   }
   c.nextSample = nextSample() // 计算下一个内存采样的距离
   mProf_Malloc(x, size) // 对本次内存分配进行采样
}
```

MemProfileRate 的默认值是 512KB，也就是默认情况下，Go 一直都有在进行内存分配的采样。`nextSample()` 用于计算下一次采样，需要等待分配多少字节的内存，把每次的内存分配顺序排列如下图所示：

![GoHeapProfile01.png#center#B#shadow](https://km.woa.com/asset/8b7d790b966d4c6e9e3946e5400f5429?height=139&width=344)

如果 c.nextSample 一直是 512KB，也就是按**等距离采样**，那么对于 >=512KB 的内存分配将每次都会被采样到，而对于 <512KB 的内存分配，越小被采样到的概率也越小。这会带来的问题就是，采样到的数据无法发现哪些内存分配比较小，而次数比较多的调用栈。

如果不使用 c.nextSample 的方式，直接进行**简单随机采样**，比如每次内存分配被采样到的概率为 1/100，那么内存分配次数多的，被采样到的样本就越多，反之越少。由于每次分配内存的字节数大小并不是一样的，我们可以从采样到的数据去评估整体每个调用栈的 alloc_objects/inuse_objects，但无法评估整体的每个调用栈的 alloc_bytes/inuse_bytes。

我们看下 `nextSample()` 是如何实现的：

```go
// nextSample 返回堆分析的下一个采样点。
// 目标是平均每 MemProfileRate 字节采样一次内存分配，且每次采样在内存分配的时间线上是完全随机分布的；
// 这对应于具有参数 MemProfileRate 的泊松过程。
// 在泊松过程中，两个样本之间的距离遵循指数分布（exp（MemProfileRate）），
// 因此最佳返回值是从平均值为 MemProfileRate 的指数分布中提取的随机数。
func nextSample() uintptr {
   if MemProfileRate == 1 {
      // 每间隔 1 个字节就要采样一次，相当于要采集所有内存分配操作
      // 所以直接返回 0 即可
      return 0
   }
   if GOOS == "plan9" {
      // Plan 9 doesn't support floating point in note handler.
      if g := getg(); g == g.m.gsignal {
         return nextSampleNoFP()
      }
   }
   // fastexprand 返回一个服从均值为 MemProfileRate 的指数分布的随机数
   // 即下一个内存采样点的距离
   return uintptr(fastexprand(MemProfileRate))
}
```

从源码注释中，可知 `nextSample()` 的实现使用了**概率论**相关的知识点，我们可以认为 `nextSample()` 的实现是为了解决这样一个问题：程序启动时，已分配的内存字节数为 0，已执行的内存分配次数也是 0，每次申请分配的内存大小至少是 1 个字节，期望平均每间隔 MemProfileRate 的字节数，对内存分配进行一次采样，求实现一个函数返回每次采样的间隔。

接下来推导问题中所描述的内存分配过程是一个强度为 1/MemProfileRate 的**泊松过程**。

根据**计数过程**的定义，我们把内存分配的总字节数看作 t， N(t) 表示已分配内存 t 字节时，发生的内存分配次数，很显然内存分配过程就是一个计数过程。

接下来参考《概率论与数理统计》P310 对**泊松过程**的定义，推导内存分配过程也是泊松过程：

将增量 ![](/assets/img/golang-pprof-sampling-00.png)，表示在内存分配的总字节数间隔 ![](/assets/img/golang-pprof-sampling-03.png) 内出现的内存分配次数。“在 ![](/assets/img/golang-pprof-sampling-03.png) 内出现 k 次内存分配”，即 ![](/assets/img/golang-pprof-sampling-01.png) 是一事件，其概率记为：

![](/assets/img/golang-pprof-sampling-02.png)

同时 `N(t)` 满足如下条件：
- 在互不重叠的区间上，状态的增量是相互独立的。比如 ![](/assets/img/golang-pprof-sampling-04.png) 这 n 个区间上发生的内存分配次数是相互独立的
- 在很小的一个区间内 ![](/assets/img/golang-pprof-sampling-05.png) 发生 1 次内存分配是有可能的，比如程序只申请了 1 个字节的内存
- 在很小的一个区间内 ![](/assets/img/golang-pprof-sampling-05.png) 发生 ≥2 次内存分配可以认为是不可能的，比如 t 增长了 1 个字节，这只可能来自 1 次内存分配
-  `N(0)=0`，内存分配的总字节数为 0，自然发生的内存分配次数也为 0

以上 4 个条件一一对应于泊松过程的定义（概率论与数理统计 P310），那么我们可以认为内存分配的整个过程可以表示为： ![](/assets/img/golang-pprof-sampling-06.png) 是强度为 λ 的泊松过程。

接下来推导 λ 的值：

由定理：强度为 λ 的泊松过程的点间间距是相互独立的随机变量，且都服从速率为 λ 的**指数分布** ![](/assets/img/golang-pprof-sampling-07.png)。即每次发生内存分配的间隔服从指数分布。

要从指数分布进行抽样，根据**逆变换法**（概率论与数理统计 P378）可得：

采样样的内存分配间隔是随机变量 ![](/assets/img/golang-pprof-sampling-08.png)

其中：
- 随机变量 U 在区间 `(0,1)` 上服从均匀分布，即 U 可以取 `(0,1)` 上的随机数。
-  T 同样服从速率 λ 的指数分布，而 T 的均值（每次采样之间的间隔的均值）为 ![](/assets/img/golang-pprof-sampling-09.png)，则 ![](/assets/img/golang-pprof-sampling-10.png)

那么我们可以用如下代码来实现 `nextSample()`：

```go
func nextSample(memProfileRate) int {
	u := rand.Intn(10000)/10000 // 产生 [0, 1) 之间的随机数
	return -1 * math.Exp(u) * memProfileRate
}
```

对比下 Go 中实际的实现：

```go
// 返回一个服从均值为 mean 的指数分布的随机数
// 即返回一个随机间隔的大小
func fastexprand(mean int) int32 {
	// Avoid overflow. Maximum possible step is
	// -ln(1/(1<<randomBitCount)) * mean, approximately 20 * mean.
	// 避免下面的计算溢出，限制了最大的间隔，实际大约是 18*mean 才对，大约是 2Gi
	switch {
	case mean > 0x7000000:
		mean = 0x7000000
	case mean == 0:
		return 0
	}

	// Take a random sample of the exponential distribution exp(-mean*x).
	// The probability distribution function is mean*exp(-mean*x), so the CDF is
	// p = 1 - exp(-mean*x), so        这里注释是错误的，应该是：p = 1 - exp(-x/mean)
	// q = 1 - p == exp(-mean*x)       这里注释是错误的，应该是：q = 1 - p == exp(-x/mean)
	// log_e(q) = -mean*x              这里注释是错误的，应该是：log_e(q) = -x/mean
	// -log_e(q)/mean = x              这里注释是错误的，应该是：x = -log_e(q) * mean
	// x = -log_e(q) * mean    为了计算效率，把 -log_e(q) 转换为 log_2(q) * (-log_e(2))
	// x = log_2(q) * (-log_e(2)) * mean    ; Using log_2 for efficiency
	const randomBitCount = 26               // 这里没有直接取 q 为 [0, 1) 内的随机数
											// 但实际上是等价的，把 log_2(x) 的图像画出来就知道了
	q := fastrandn(1<<randomBitCount) + 1         // q 的取值范围：[1, 2^26]
	qlog := fastlog2(float64(q)) - randomBitCount // qlog 的取值范围：[-26, 0]
	if qlog > 0 { // 这个判断似乎是多余的
		qlog = 0
	}
	const minusLog2 = -0.6931471805599453 // -ln(2) 的值
	// qlog*minusLog2 的取值范围：[0, 18.021826694558577]
	return int32(qlog*(minusLog2*float64(mean))) + 1 // 返回值范围：[1, 18*mean+1]
}
```

在收集了采样数据写入文件的时候，还需要对数据进行调整，以获得对实际情况的内存分配次数和分配字节数的整体评估。对采样数据的调整在 `runtime/pprof/protomem.go` 中的方法 scaleHeapSample，由 writeHeapProto 调用：

采样一次 size 大小的内存分配的概率由指数分布的**概率分布函数**得出： ![](/assets/img/golang-pprof-sampling-11.png)，可以理解为 `nextSample()` 返回的随机数小于等于 size 的概率。

```go
// scaleHeapSample adjusts the data from a heap Sample to
// account for its probability of appearing in the collected
// data. heap profiles are a sampling of the memory allocations
// requests in a program. We estimate the unsampled value by dividing
// each collected sample by its probability of appearing in the
// profile. heap profiles rely on a poisson process to determine
// which samples to collect, based on the desired average collection
// rate R. The probability of a sample of size S to appear in that
// profile is 1-exp(-S/R).
func scaleHeapSample(count, size, rate int64) (int64, int64) {
	if count == 0 || size == 0 {
		return 0, 0
	}

	if rate <= 1 {
		// if rate==1 all samples were collected so no adjustment is needed.
		// if rate<1 treat as unknown and skip scaling.
		return count, size
	}

	// 计算当前调用栈平均采样的字节数
	avgSize := float64(size) / float64(count)
	// (1 - math.Exp(-avgSize/float64(rate))) 即为采样到一次内存分配为 avgSize 字节的概率
	scale := 1 / (1 - math.Exp(-avgSize/float64(rate)))

	// 返回当前调用栈整体的内存分配次数和总字节数
	return int64(float64(count) * scale), int64(float64(size) * scale)
}
```

我们可以修改 `runtime.MemProfileRate` 的值来调整 `fastexprand(MemProfileRate)` 产生的随机数，通常不需要调整，但如果我们担心某些调用栈分配的内存字节数比较小，且分配的次数也相对很少，希望能尽量被采样到时，可以调小 `runtime.MemProfileRate` 的值，如下是不同 MemProfileRate 对于不同内存分配大小的采样概率：

```text
MemProfileRate 2^3byte  2^4byte  2^5byte  ...内存分配的大小...2^8byte...                                                                                                                                                                                ...2^29byte
64KB           0.000122 0.000244 0.000488 0.000976 0.001951 0.003899 0.007782 0.015504 0.030767 0.060587 0.117503 0.221199 0.393469 0.632121 0.864665 0.981684 0.999665 1.000000 1.000000 1.000000 1.000000 1.000000 1.000000 1.000000 1.000000 1.000000 1.000000
128KB          0.000061 0.000122 0.000244 0.000488 0.000976 0.001951 0.003899 0.007782 0.015504 0.030767 0.060587 0.117503 0.221199 0.393469 0.632121 0.864665 0.981684 0.999665 1.000000 1.000000 1.000000 1.000000 1.000000 1.000000 1.000000 1.000000 1.000000
256KB          0.000031 0.000061 0.000122 0.000244 0.000488 0.000976 0.001951 0.003899 0.007782 0.015504 0.030767 0.060587 0.117503 0.221199 0.393469 0.632121 0.864665 0.981684 0.999665 1.000000 1.000000 1.000000 1.000000 1.000000 1.000000 1.000000 1.000000
512KB          0.000015 0.000031 0.000061 0.000122 0.000244 0.000488 0.000976 0.001951 0.003899 0.007782 0.015504 0.030767 0.060587 0.117503 0.221199 0.393469 0.632121 0.864665 0.981684 0.999665 1.000000 1.000000 1.000000 1.000000 1.000000 1.000000 1.000000
1024KB         0.000008 0.000015 0.000031 0.000061 0.000122 0.000244 0.000488 0.000976 0.001951 0.003899 0.007782 0.015504 0.030767 0.060587 0.117503 0.221199 0.393469 0.632121 0.864665 0.981684 0.999665 1.000000 1.000000 1.000000 1.000000 1.000000 1.000000
2048KB         0.000004 0.000008 0.000015 0.000031 0.000061 0.000122 0.000244 0.000488 0.000976 0.001951 0.003899 0.007782 0.015504 0.030767 0.060587 0.117503 0.221199 0.393469 0.632121 0.864665 0.981684 0.999665 1.000000 1.000000 1.000000 1.000000 1.000000
4096KB         0.000002 0.000004 0.000008 0.000015 0.000031 0.000061 0.000122 0.000244 0.000488 0.000976 0.001951 0.003899 0.007782 0.015504 0.030767 0.060587 0.117503 0.221199 0.393469 0.632121 0.864665 0.981684 0.999665 1.000000 1.000000 1.000000 1.000000
```

在 `mProf_Malloc(p unsafe.Pointer, size uintptr)` 中会对调用栈对应的计数桶累加分配次数和分配的字节数，同时通过 `setprofilebucket(p unsafe.Pointer, b *bucket)` 在 `mspan.specials` 上标记内存地址 p 在分配的时候被采样了。

```go
// Called by malloc to record a profiled block.
func mProf_Malloc(p unsafe.Pointer, size uintptr) {
   var stk [maxStack]uintptr
   nstk := callers(4, stk[:])
   lock(&proflock)
   b := stkbucket(memProfile, size, stk[:nstk], true)
   c := mProf.cycle
   mp := b.mp()
   mpc := &mp.future[(c+2)%uint32(len(mp.future))]
   mpc.allocs++
   mpc.alloc_bytes += size
   unlock(&proflock)

   // Setprofilebucket locks a bunch of other mutexes, so we call it outside of proflock.
   // This reduces potential contention and chances of deadlocks.
   // Since the object must be alive during call to mProf_Malloc,
   // it's fine to do this non-atomically.
   systemstack(func() {
      setprofilebucket(p, b)
   })
}
```

在触发 mspan 的清理时（时机有很多），如果 mspan 上的 object 没有被 GC 标记为存活，且 object 有 profile 类型的 special，则会调用 `mProf_Free(b *bucket, size uintptr)` 累计释放对象的数量，以及释放的字节数：

```
// Called when freeing a profiled block.
func mProf_Free(b *bucket, size uintptr) {
   lock(&proflock)
   c := mProf.cycle
   mp := b.mp()
   mpc := &mp.future[(c+1)%uint32(len(mp.future))]
   mpc.frees++
   mpc.free_bytes += size
   unlock(&proflock)
}
```

那么 inuse_objects 和 inuse_space 就可以计算出来：

```go
// InUseBytes returns the number of bytes in use (AllocBytes - FreeBytes).
func (r *MemProfileRecord) InUseBytes() int64 { return r.AllocBytes - r.FreeBytes }

// InUseObjects returns the number of objects in use (AllocObjects - FreeObjects).
func (r *MemProfileRecord) InUseObjects() int64 {
	return r.AllocObjects - r.FreeObjects
}
```

# Block Profiler

Block Profiler 能帮助我们分析哪些调用栈因为阻塞而等待的时长分布情况，会采样如下阻塞事件等待的时长：

- chansend：当向 channel 发送数据而阻塞时
- chanrecv：当从 channel 接收数据而阻塞时
- select：当 select 的 channel 都没有就绪时，且没有 default 分支，而阻塞时
- semacquire1：请求获取锁而阻塞时：
    - sync 包：sync.Mutex.Lock、sync.RWMutex.RLock、sync.RWMutex.Lock、sync.WaitGroup.Wait
    - internal/poll包：poll.fdMutex.rwlock、poll.FD.Close
- notifyListWait：等通知而阻塞时（sync.Cond.Wait）

默认情况下，不会进行 Block Profiler 的采样，可以通过设置 `runtime.SetBlockProfileRate(rate int)` 开启，rate 的单位为 ns。我们可以通过如下 API 来获得一份阻塞时长的采样结果：

- 通过命令行测试工具：`go test -blockprofile block.pprof`
- 代码中获取：`pprof.Lookup("block").WriteTo(w, 0)`
- 通过 HTTP 接口：`import _ "net/http/pprof"` 和 `GET /debug/pprof/block?seconds=30` 如果传了 seconds=30 参数，用于获取 30s 内的增量采样数据，否则获取全量采样数据

在需要采样的地方都会调用 blockevent：

```go
// cycles 阻塞等待时长，单位为 CPU Tick
// skip 生成调用栈时跳过的栈帧数量
func blockevent(cycles int64, skip int) {
	if cycles <= 0 {
		cycles = 1
	}

	rate := int64(atomic.Load64(&blockprofilerate))
	if blocksampled(cycles, rate) {
		saveblockevent(cycles, rate, skip+1, blockProfile)
	}
}

// blocksampled 使用的是 PPS 采样（概率比例规模采样）
// 当 cycles >= rate 时，采样是全采样
// 当 cycles < rate 时，被采样到的概率为 cycles/rate
func blocksampled(cycles, rate int64) bool {
	if rate <= 0 || (rate > cycles && int64(fastrand())%rate > cycles) {
		return false
	}
	return true
}
```

在 saveblockevent 中，会对 cycles < rate 的样本数据按照采样的概率进行调整：

```go
// cycles 阻塞等待时长，单位为 CPU Tick
// rate 为 blockprofilerate
// skip 生成调用栈时跳过的栈帧数量
// which 为 blockProfile
func saveblockevent(cycles, rate int64, skip int, which bucketType) {
	gp := getg()
	// 获取当前调用栈
	var nstk int
	var stk [maxStack]uintptr
	if gp.m.curg == nil || gp.m.curg == gp {
		nstk = callers(skip, stk[:])
	} else {
		nstk = gcallers(gp.m.curg, skip, stk[:])
	}
	lock(&proflock)
	// 调用栈对应的计数桶
	b := stkbucket(which, 0, stk[:nstk], true)

	if which == blockProfile && cycles < rate {
		// Remove sampling bias, see discussion on http://golang.org/cl/299991.
		b.bp().count += float64(rate) / float64(cycles) // 相当于 1/(cycles/rate)
		b.bp().cycles += rate                           // 相当于 cycles/(cycles/rate)
	} else {
		b.bp().count++
		b.bp().cycles += cycles
	}
	unlock(&proflock)
}
```

因为已经在 `saveblockevent()` 中对部分采样的样本数据进行了调整，所以在写入采样数据文件时就不需要再做调整了。

# Mutex Profiler

Mutex Profiler 能帮助我们分析调用栈持有锁时长的分布情况，会采样如下锁的持有时长：

- semrelease1：sync.Mutex.Unlock、sync.RWMutex.RUnlock、sync.RWMutex.Unlock

默认情况下，不会进行 Mutex Profiler 的采样，可以通过设置 `runtime.SetMutexProfileFraction(rate int)` 开启，使用**简单随机采样**，每个事件被采样到的概率为 1/rate。我们可以通过如下 API 来获得一份锁持有时长的采样结果：

- 通过命令行测试工具：`go test -mutexprofile block.pprof`
- 代码中获取：`pprof.Lookup("mutex").WriteTo(w, 0)`
- 通过 HTTP 接口：`import _ "net/http/pprof"` 和 `GET /debug/pprof/mutex?seconds=30` 如果传了 seconds=30 参数，用于获取 30s 内的增量采样数据，否则获取全量采样数据

在需要采样的地方都会调用 mutexevent：

```go
// cycles 阻塞等待时长，单位时 CPU Tick
// skip 生成调用栈时跳过的栈帧数量
func mutexevent(cycles int64, skip int) {
	if cycles < 0 {
		cycles = 0
	}
	rate := int64(atomic.Load64(&mutexprofilerate))
	// TODO(pjw): measure impact of always calling fastrand vs using something
	// like malloc.go:nextSample()
	if rate > 0 && int64(fastrand())%rate == 0 { // 1/rate 概率采样
		saveblockevent(cycles, rate, skip+1, mutexProfile)
	}
}

func saveblockevent(cycles, rate int64, skip int, which bucketType) {
	gp := getg()
	var nstk int
	var stk [maxStack]uintptr
	if gp.m.curg == nil || gp.m.curg == gp {
		nstk = callers(skip, stk[:])
	} else {
		nstk = gcallers(gp.m.curg, skip, stk[:])
	}
	lock(&proflock)
	b := stkbucket(which, 0, stk[:nstk], true)

	if which == blockProfile && cycles < rate {
		// Remove sampling bias, see discussion on http://golang.org/cl/299991.
		b.bp().count += float64(rate) / float64(cycles)
		b.bp().cycles += rate
	} else {
	    // mutexProfile 直接累加采样到的数据
		b.bp().count++
		b.bp().cycles += cycles
	}
	unlock(&proflock)
}
```

在写入采样数据文件时，会对采样到的数据进行调整，以获得对整体情况的评估：

```go
func scaleMutexProfile(cnt int64, ns float64) (int64, float64) {
	period := runtime.SetMutexProfileFraction(-1) // 返回 mutexprofilerate
	return cnt * int64(period), ns * float64(period)
}
```

但正如 Memory Profile 中提到的，每个事件的锁持有时长是不一样的，Mutex Profile 使用的简单随机采样方法，可以用来评估整体每个调用栈发生的锁事件个数，但评估整体的每个调用栈的锁时长可能会有比较大的偏差。

# Goroutine Profiler

Goroutine Profiler 能帮助我们分析哪些调用栈的协程数比较多，每个协程当前的状态（Running、IO wait等等），我们可以通过如下 API 来获得一份协程的分析结果：

- 代码中获取：`pprof.Lookup("goroutine").WriteTo(w, deubg)`
    - debug 取值为：
        - 0：生成 protocol buffer 格式按调用栈聚合的数据
        - 1：生成 text 格式按调用栈聚合的数据
        - 2：生成 text 格式，会输出每个协程详细信息的数据
- 通过 HTTP 接口：`import _ "net/http/pprof"` 和 `GET /debug/pprof/goroutine?debug=1|2`，也可以传 seconds 参数（不能跟 debug 参数一起使用），用于获取增量的数据

当 debug 为 0/1 时，生成协程的分析数据：

- writeRuntimeProfile(w, debug, "goroutine", runtime_goroutineProfileWithLabels)
    - goroutineProfileWithLabels(p, labels)
        - stopTheWorld("profile")
        - forEachGRace(func(gp1 \*g)
            - 如果 gp1 的状态不是 \_Gdead ，且不属于系统协程
            - saveg(\^uintptr(0), ^uintptr(0), gp1, &r[0]) 记录 gp1 当前的调用栈
        - startTheWorld()

当 debug 为 2 时，生成协程的分析数据：

- writeGoroutineStacks(w)
    - runtime.Stack(buf, true)
        - stopTheWorld("stack trace")
        - forEachGRace(func(gp \*g)
            - 如果 gp 的状态不是 \_Gdead ，且不属于系统协程，且 traceback level >=2
            - goroutineheader(gp) 记录 gp 的协程 ID、当前的状态
            - traceback(\^uintptr(0), \^uintptr(0), 0, gp) 记录 gp 当前的调用栈
        - startTheWorld()

# 采样方法对比

Memory Profile、Block Profile、Mutex Profile 这三种类型采样的数据是类似的，但却使用了三种不同的采样方法，其中 Mutex Profile 使用的**简单随机采样**是最简单的，但采样到的数据只能用于评估整体每个调用栈的事件发生的次数。而 Memory Profile 和 Block Profile 的采样方法都把事件的属性（时长、分配的内存字节数）考虑进去了，采样到的数据不仅可以评估整体每个调用栈的事件发生的次数，还可以评估整体每个调用栈的事件属性的分布情况。

以下修改 Go1.18.4 Memory Profile 的采样方法为 PPS 采样，与指数分布采样进行对比：

测试代码：

```go
package main

import (
	"log"
	"os"
	"runtime/pprof"
)

func main() {
	allocate_large_small()
	allocate_small()

	w, err := os.Create("heap.prof")
	if err != nil {
		log.Fatal(err)
	}
	err = pprof.Lookup("allocs").WriteTo(w, 0)
	if err != nil {
		log.Fatal(err)
	}
	w.Close()
}

var a16 *[16]byte
var a512 *[512]byte
var a256 *[256]byte
var a1k *[1024]byte
var a256k *[256 * 1024]byte
var a512k *[512 * 1024]byte

func allocate_large_small() {
	// 10w * 9
	for i := 0; i < 100000; i++ {
		a512k = new([512 * 1024]byte)
		a256k = new([256 * 1024]byte)
		a1k = new([1024]byte)
		a256k = new([256 * 1024]byte)
		a512 = new([512]byte)
		a256k = new([256 * 1024]byte)
		a256 = new([256]byte)
		a256k = new([256 * 1024]byte)
		a16 = new([16]byte)
	}
}

func allocate_small() {
	// 100w * 4
	for i := 0; i < 1000000; i++ {
		a1k = new([1024]byte)
		a512 = new([512]byte)
		a256 = new([256]byte)
		a16 = new([16]byte)
	}
}
```

alloc_objects 对比：`go tool pprof -lines -top -alloc_objects heap heap.prof`

| Line | 指数分布采样 | PPS采样 | 实际值
| :---: | :---: | :---: | :---: |
| heap.go:52 | 950286 | 1343488 | 100w |
| heap.go:51 | 1022201 | 1019904 | 100w |
| heap.go:50 | 1014255 | 1005568 | 100w |
| heap.go:49 | 944537 | 1014272 | 100w |
| heap.go:42 | 131074 | 98304 | 10w |
| heap.go:41 | 99883 | 99290 | 10w |
| heap.go:40 | 133152 | 102400 | 10w |
| heap.go:39 | 99773 | 100052 | 10w |
| heap.go:38 | 97327 | 102400 | 10w |
| heap.go:37 | 100239 | 100616 | 10w |
| heap.go:36 | 80975 | 93696 | 10w |
| heap.go:35 | 100117 | 100188 | 10w |
| heap.go:34 | 100287 | 100000 | 10w |

alloc_space 对比：`go tool pprof -lines -nodefraction 0 -edgefraction 0 -top -alloc_space heap heap.prof`

| Line | 指数分布采样 | PPS采样 | 实际值
| :---: | :---: | :---: | :---: |
| heap.go:34 | 50143.92MB | 50000MB | 50000MB |
| heap.go:35 | 25029.27MB | 25047MB | 25000MB |
| heap.go:37 | 25059.77MB | 25154MB | 25000MB |
| heap.go:39 | 24943.49MB | 25013MB | 25000MB |
| heap.go:41 | 24970.81MB | 24822.50MB | 25000MB |
| heap.go:49 | 922.40MB | 990.50MB | 976.5625MB |
| heap.go:50 | 495.24MB | 491MB | 488.28125MB |
| heap.go:51 | 249.56MB | 249MB | 244.140625MB |
| heap.go:36 | 79.08MB | 91.50MB | 97.65625MB |
| heap.go:38 | 47.52MB | 50MB | 48.828125MB |
| heap.go:40 | 32.51MB | 25MB | 24.4140625 |
| heap.go:52 | 14.50MB | 20.50MB | 15.2587890625MB |
| heap.go:42 | 2MB | 1.50MB | 1.52587890625MB |

对比发现，指数分布采样和 PPS 采样效果差不多，那么为何还要使用“复杂”的指数分布采样呢？在 `mutexevent()` 的代码注释中有提到建议使用 `nextSample()` 的采样方法来代替简单随机采样，以减少对性能的影响。

指数分布采样的计算耗时在于 `fastexprand(mean int) int32`，而 PPS 采样的计算耗时在于 `fastrand() uint32`，跑下单测对比下性能：

> Go 源码中的单测怎么跑：
> cd src && sh make.bash --dist-tool
> $GOTOOLDIR/dist test -run runtime

```
// BenchmarkFastexprand 单测在源码中是没有的，需要自己写下
BenchmarkFastexprand/2-12       193027530                6.180 ns/op
BenchmarkFastexprand/3-12       190793637                6.362 ns/op
BenchmarkFastexprand/4-12       192437007                6.302 ns/op
BenchmarkFastexprand/5-12       190947202                6.254 ns/op

BenchmarkFastrand/2-12          871177311                1.350 ns/op
BenchmarkFastrand/3-12          891198306                1.367 ns/op
BenchmarkFastrand/4-12          884950987                1.390 ns/op
BenchmarkFastrand/5-12          886145552                1.387 ns/op
```

再看下这两个函数所需的调用次数和采样到的样本数：

- `fastexprand(mean int) int32` 调用次数 224480，采样到的样本 224480 个
- `fastrand() uint32` 调用次数 4800000，采样到的样本 304259 个（其中 100000 个样本是 512KB 的内存分配次数）

对比之后可以发现，虽然 PPS 采样使用的 `fastrand() uint32` 大约比指数分布采样使用的 `fastexprand(mean int) int32` 快 5 倍，但是调用次数却多了约 20 倍，且每次内存分配大小达到 MemProfileRate 都会被采样到，每采样一个样本，就需要生成一次当前的调用栈，所以综合来看，指数分布的采样方法性能是要优于 PPS 采样方法的。

# 参考资料

- [DataDog/go-profiler-notes](https://github.com/DataDog/go-profiler-notes)
- [Profiling Improvements in Go 1.18](https://felixge.de/2022/02/11/profiling-improvements-in-go-1.18/)
- [Linux 系统调用 rt_sigaction](https://linux.die.net/man/2/rt_sigaction)
- [Linux 信号](https://man7.org/linux/man-pages/man7/signal.7.html)
- 《概率论与数理统计》第四版 浙江大学
- [B站 泊松过程](https://www.bilibili.com/video/BV1dt4y1k7rn/?p=1&vd_source=bb14f15577931267467de5a5d901bb35)
- [泊松过程两种定义等价性证明](https://www.docin.com/p-1309509690.html)
- [B站 泊松分布和指数分布](https://www.bilibili.com/video/BV1LG411L7qt/?spm_id_from=333.337.search-card.all.click&vd_source=bb14f15577931267467de5a5d901bb35)
- [逆变换采样](https://www.cnblogs.com/mingyangovo/p/14668731.html)