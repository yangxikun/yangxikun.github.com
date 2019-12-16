---
layout: post
title: "golang goroutine 堆栈"
description: "golang goroutine 堆栈"
category: GoLang
tags: []
---
{% include JB/setup %}

参考文章：

* [Contiguous stacks in Go](https://agis.io/post/contiguous-stacks-golang/)
* [Contiguous stacks](https://docs.google.com/document/d/1wAaf1rYoM4S4gtnPh0zOlGzWtrZFQ5suE8qr2sD8uWQ/pub)
* [聊一聊goroutine stack](https://kirk91.github.io/posts/2d571d09/)

#### 栈相关的数据结构

```go
// Stack describes a Go execution stack.
// The bounds of the stack are exactly [lo, hi),
// with no implicit data structures on either side.
// stack 数据结构用于表示一个 goroutine 的执行堆栈
// 堆栈的边界范围是 [lo, hi)
type stack struct {
    lo uintptr
    hi uintptr
}

type g struct {
    // Stack parameters.
    // stack describes the actual stack memory: [stack.lo, stack.hi).
    // stackguard0 is the stack pointer compared in the Go stack growth prologue.
    // It is stack.lo+StackGuard normally, but can be StackPreempt to trigger a preemption.
    // stackguard1 is the stack pointer compared in the C stack growth prologue.
    // It is stack.lo+StackGuard on g0 and gsignal stacks.
    // It is ~0 on other goroutine stacks, to trigger a call to morestackc (and crash).
    stack       stack   // offset known to runtime/cgo
    // stackguard0 用于跟函数的 sp 进行比较，通常等于 stack.lo+StackGuard，但是在需要触发抢占调度时，会被赋值为 StackPreempt
    stackguard0 uintptr // offset known to liblink
    // stackguard1 用于跟系统线程的 sp 进行比较，在 g0 和 gsignal 上等于 stack.lo+StackGuard，在其他 goroutine 上等于 ~0，用于触发栈扩容和崩溃
    stackguard1 uintptr // offset known to liblink
    ......
}
```

<!--more-->

#### 栈的分配

> goroutine 的栈是在堆内存上分配的！！！

```go
// stackalloc allocates an n byte stack.
//
// stackalloc must run on the system stack because it uses per-P
// resources and must not split the stack.
//
//go:systemstack
func stackalloc(n uint32) stack {
    // Stackalloc must be called on scheduler stack, so that we
    // never try to grow the stack during the code that stackalloc runs.
    // Doing so would cause a deadlock (issue 1547).
    // 因为是在 systemstack 上执行的，当前 g 必须是 g0
    thisg := getg()
    if thisg != thisg.m.g0 {
        throw("stackalloc not on scheduler stack")
    }
    if n&(n-1) != 0 {
        throw("stack size not a power of 2")
    }
    if stackDebug >= 1 {
        print("stackalloc ", n, "\n")
    }

    if debug.efence != 0 || stackFromSystem != 0 {
        n = uint32(round(uintptr(n), physPageSize))
        v := sysAlloc(uintptr(n), &memstats.stacks_sys)
        if v == nil {
            throw("out of memory (stackalloc)")
        }
        return stack{uintptr(v), uintptr(v) + uintptr(n)}
    }

    // Small stacks are allocated with a fixed-size free-list allocator.
    // If we need a stack of a bigger size, we fall back on allocating
    // a dedicated span.
    var v unsafe.Pointer
    if n < _FixedStack<<_NumStackOrders && n < _StackCacheSize {
        // 需要分配的栈空间比较小
        // order 用于索引对应大小的栈空间
        order := uint8(0)
        n2 := n
        for n2 > _FixedStack {
            order++
            n2 >>= 1
        }
        var x gclinkptr
        c := thisg.m.mcache
        // 如果
        // 1. 不使用 P 的空闲栈空间缓存
        // 2. 当前 m 没有关联的 mcache
        // 3. thisg.m.preemptoff 不为空，好像跟 gc 相关
        // 则从全局的空闲栈空间池子中分配
        if stackNoCache != 0 || c == nil || thisg.m.preemptoff != "" {
            // c == nil can happen in the guts of exitsyscall or
            // procresize. Just get a stack from the global pool.
            // Also don't touch stackcache during gc
            // as it's flushed concurrently.
            lock(&stackpoolmu)
            x = stackpoolalloc(order)
            unlock(&stackpoolmu)
        } else {
            // 从 P 的栈空间缓存中分配
            x = c.stackcache[order].list
            if x.ptr() == nil {
                stackcacherefill(c, order)
                x = c.stackcache[order].list
            }
            c.stackcache[order].list = x.ptr().next
            c.stackcache[order].size -= uintptr(n)
        }
        v = unsafe.Pointer(x)
    } else {
        // 需要分配的栈空间比较大，从全局的大栈空间池子中分配
        var s *mspan
        npage := uintptr(n) >> _PageShift
        log2npage := stacklog2(npage)

        // Try to get a stack from the large stack cache.
        lock(&stackLarge.lock)
        if !stackLarge.free[log2npage].isEmpty() {
            s = stackLarge.free[log2npage].first
            stackLarge.free[log2npage].remove(s)
        }
        unlock(&stackLarge.lock)

        if s == nil {
            // Allocate a new stack from the heap.
            s = mheap_.allocManual(npage, &memstats.stacks_inuse)
            if s == nil {
                throw("out of memory")
            }
            osStackAlloc(s)
            s.elemsize = uintptr(n)
        }
        v = unsafe.Pointer(s.base())
    }

    if raceenabled {
        racemalloc(v, uintptr(n))
    }
    if msanenabled {
        msanmalloc(v, uintptr(n))
    }
    if stackDebug >= 1 {
        print("  allocated ", v, "\n")
    }
    return stack{uintptr(v), uintptr(v) + uintptr(n)}
}
```

Linux 进程的内存布局：

参考文章：[Understanding the Memory Layout of Linux Executables](https://gist.github.com/CMCDragonkai/10ab53654b2aa6ce55c11cfc5b2432a4)

```text
High Address   +------------------------------------------------------------+
               |                                                            |
      ^        | Kernel Space                                               |
      |        |                                                            |
      |        +------------------------------------------------------------+
      |        |                              |                             |
      |        |                              |                             |
      |        | User Stack                   |                             |
      |        |                              |                             |
      |        |                              v                             |
      |        +------------------------------------------------------------+
      |        |                                                            |
      |        | Memory Mapped Region for Shared Libraries or Anything Else |
      |        |                                                            |
      |        +------------------------------------------------------------+
      |        |                              ^                             |
      |        | Heap                         |                             |
      |        |                              |                             |
      |        +------------------------------------------------------------+
      |        |                                                            |
      |        | Uninitialised Data (.bss)                                  |
      |        |                                                            |
      |        +------------------------------------------------------------+
      |        |                                                            |
      |        | Initialised Data (.data)                                   |
      |        |                                                            |
      |        +------------------------------------------------------------+
      |        |                                                            |
      |        | Program Text (.text)                                       |
      |        |                                                            |
      |        +------------------------------------------------------------+
      |        |                                                            |
               |                                                            |
 Low Address   +------------------------------------------------------------+
```

栈结构在内存上的图示：

```text
         High Address  +---------------+ stack.hi                    
                       |               |                             
              |        |               |                             
              |        |               |                             
              |        |               |                             
              |        |               |                             
              |        |               |                             
              |        |               |                             
 stack grow direction  |               |                             
              |        |               | g.stackguard0               
              |        |               |     ^      ^                
              |        |               |     |      |                
              |        |               |     v      |                
              |        |               | StackSmall | StackGuard     
              |        |               |            |                
              |        |               |            |                
              v        |               |            |                
                       |               |            v                
         Low Address   +---------------+ stack.lo
```

#### 栈的扩容

go 在进行目标代码生成的时候（src/cmd/internal/obj/x86/obj6.go:stacksplit）会根据函数栈帧大小插入相应的指令，检查当前 goroutine 的栈空间是否足够。有如下几种情况：

1、当函数是叶子节点，且栈帧小于等于 112 字节时，不会插入检查指令

2、当叶子函数栈帧大小为 \[120字节, 128字节\] 或非叶子函数栈帧大小为 (0, 128字节]

> SP <= stackguard

```text
0x0000 00000 (stack.go:8)    MOVQ    (TLS), CX
0x0009 00009 (stack.go:8)    CMPQ    SP, 16(CX) ;
0x000d 00013 (stack.go:8)    JLS    41
......
0x0029 00041 (stack.go:8)    CALL    runtime.morestack_noctxt(SB)
0x002e 00046 (stack.go:8)    JMP    0
```

3、当函数栈帧大小为 (128字节, 4096字节]

> SP-framesize <= stackguard-StackSmall

```text
0x0000 00000 (stack.go:31)    MOVQ    (TLS), CX
0x0009 00009 (stack.go:31)    LEAQ    -3968(SP), AX
0x0011 00017 (stack.go:31)    CMPQ    AX, 16(CX) ;
0x0015 00021 (stack.go:31)    JLS    93
......
0x005d 00093 (stack.go:31)    CALL    runtime.morestack_noctxt(SB)
0x0062 00098 (stack.go:31)    JMP    0
```

4、当函数栈帧大小 > 4096字节

> SP-stackguard+StackGuard <= framesize + (StackGuard-StackSmall)

```text
0x0000 00000 (stack.go:36)    MOVQ    (TLS), CX
0x0009 00009 (stack.go:36)    MOVQ    16(CX), SI ;
0x000d 00013 (stack.go:36)    CMPQ    SI, $-1314
0x0014 00020 (stack.go:36)    JEQ    102
0x0016 00022 (stack.go:36)    LEAQ    880(SP), AX
0x001e 00030 (stack.go:36)    SUBQ    SI, AX
0x0021 00033 (stack.go:36)    CMPQ    AX, $4856
0x0027 00039 (stack.go:36)    JLS    102
......
0x0066 00102 (stack.go:36)    CALL    runtime.morestack_noctxt(SB)
0x006b 00107 (stack.go:36)    JMP    0
```

2、3、4 的计算主要是为了确保 SP - framesize <= stackguard - StackSmall。根据栈帧大小，通过不同的方式进行计算。

因为 StackSmall = 128 bytes，所以 2 的存在是为了节省掉 3 中的减法操作。对于 4 目前并不太理解。

尽管检测的方式不同，但扩容时都是调用`runtime.morestack_noctxt()`函数，实际调用的是汇编实现的函数`runtime·morestack`：

```text
// Called during function prolog when more stack is needed.
//
// The traceback routines see morestack on a g0 as being
// the top of a stack (for example, morestack calling newstack
// calling the scheduler calling newm calling gc), so we must
// record an argument size. For that purpose, it has no arguments.
TEXT runtime·morestack(SB),NOSPLIT,$0-0
    // Cannot grow scheduler stack (m->g0).
    get_tls(CX)
    MOVQ    g(CX), BX ; 获取当前 g
    MOVQ    g_m(BX), BX ; 获取 m
    MOVQ    m_g0(BX), SI ; 获取 g0
    CMPQ    g(CX), SI ; 判断当前 g 是不是 g0
    JNE    3(PC)
    CALL    runtime·badmorestackg0(SB) ; 当前是 g0，抛出异常
    CALL    runtime·abort(SB)

    // Cannot grow signal stack (m->gsignal).
    MOVQ    m_gsignal(BX), SI
    CMPQ    g(CX), SI
    JNE    3(PC)
    CALL    runtime·badmorestackgsignal(SB) ; 当前是 gsignal，抛出异常
    CALL    runtime·abort(SB)

    // Called from f.
    // Set m->morebuf to f's caller.
    NOP    SP    // tell vet SP changed - stop checking offsets
    MOVQ    8(SP), AX    // f's caller's PC
    MOVQ    AX, (m_morebuf+gobuf_pc)(BX)
    LEAQ    16(SP), AX    // f's caller's SP
    MOVQ    AX, (m_morebuf+gobuf_sp)(BX)
    get_tls(CX)
    MOVQ    g(CX), SI
    MOVQ    SI, (m_morebuf+gobuf_g)(BX)

    // f 是需要扩容的函数
    // Set g->sched to context in f.
    MOVQ    0(SP), AX // f's PC
    MOVQ    AX, (g_sched+gobuf_pc)(SI)
    MOVQ    SI, (g_sched+gobuf_g)(SI)
    LEAQ    8(SP), AX // f's SP
    MOVQ    AX, (g_sched+gobuf_sp)(SI)
    MOVQ    BP, (g_sched+gobuf_bp)(SI)
    MOVQ    DX, (g_sched+gobuf_ctxt)(SI)

    // 在系统线程栈上调用 newstack 方法
    // Call newstack on m->g0's stack.
    MOVQ    m_g0(BX), BX
    MOVQ    BX, g(CX)
    MOVQ    (g_sched+gobuf_sp)(BX), SP
    CALL    runtime·newstack(SB)
    CALL    runtime·abort(SB)    // crash if newstack returns
    RET
```

扩容栈的操作：

```go
// Called from runtime·morestack when more stack is needed.
// Allocate larger stack and relocate to new stack.
// Stack growth is multiplicative, for constant amortized cost.
//
// g->atomicstatus will be Grunning or Gscanrunning upon entry.
// If the GC is trying to stop this g then it will set preemptscan to true.
//
// This must be nowritebarrierrec because it can be called as part of
// stack growth from other nowritebarrierrec functions, but the
// compiler doesn't check this.
//
//go:nowritebarrierrec
func newstack() {
    thisg := getg() // g0
    // TODO: double check all gp. shouldn't be getg().
    if thisg.m.morebuf.g.ptr().stackguard0 == stackFork {
        throw("stack growth after fork")
    }
    if thisg.m.morebuf.g.ptr() != thisg.m.curg {
        print("runtime: newstack called from g=", hex(thisg.m.morebuf.g), "\n"+"\tm=", thisg.m, " m->curg=", thisg.m.curg, " m->g0=", thisg.m.g0, " m->gsignal=", thisg.m.gsignal, "\n")
        morebuf := thisg.m.morebuf
        traceback(morebuf.pc, morebuf.sp, morebuf.lr, morebuf.g.ptr())
        throw("runtime: wrong goroutine in newstack")
    }

    gp := thisg.m.curg // 需要扩容栈帧的 g

    if thisg.m.curg.throwsplit { // g 不应该扩容
        // Update syscallsp, syscallpc in case traceback uses them.
        morebuf := thisg.m.morebuf
        gp.syscallsp = morebuf.sp
        gp.syscallpc = morebuf.pc
        pcname, pcoff := "(unknown)", uintptr(0)
        f := findfunc(gp.sched.pc)
        if f.valid() {
            pcname = funcname(f)
            pcoff = gp.sched.pc - f.entry
        }
        print("runtime: newstack at ", pcname, "+", hex(pcoff),
            " sp=", hex(gp.sched.sp), " stack=[", hex(gp.stack.lo), ", ", hex(gp.stack.hi), "]\n",
            "\tmorebuf={pc:", hex(morebuf.pc), " sp:", hex(morebuf.sp), " lr:", hex(morebuf.lr), "}\n",
            "\tsched={pc:", hex(gp.sched.pc), " sp:", hex(gp.sched.sp), " lr:", hex(gp.sched.lr), " ctxt:", gp.sched.ctxt, "}\n")

        thisg.m.traceback = 2 // Include runtime frames
        traceback(morebuf.pc, morebuf.sp, morebuf.lr, gp)
        throw("runtime: stack split at bad time")
    }

    morebuf := thisg.m.morebuf
    thisg.m.morebuf.pc = 0
    thisg.m.morebuf.lr = 0
    thisg.m.morebuf.sp = 0
    thisg.m.morebuf.g = 0

    // NOTE: stackguard0 may change underfoot, if another thread
    // is about to try to preempt gp. Read it just once and use that same
    // value now and below.
    preempt := atomic.Loaduintptr(&gp.stackguard0) == stackPreempt

    // Be conservative about where we preempt.
    // We are interested in preempting user Go code, not runtime code.
    // If we're holding locks, mallocing, or preemption is disabled, don't
    // preempt.
    // This check is very early in newstack so that even the status change
    // from Grunning to Gwaiting and back doesn't happen in this case.
    // That status change by itself can be viewed as a small preemption,
    // because the GC might change Gwaiting to Gscanwaiting, and then
    // this goroutine has to wait for the GC to finish before continuing.
    // If the GC is in some way dependent on this goroutine (for example,
    // it needs a lock held by the goroutine), that small preemption turns
    // into a real deadlock.
    if preempt {
        // 如果 M 上有锁
        // 如果正在进行内存分配
        // 如果明确禁止抢占
        // 如果 P 的状态不是 running
        if thisg.m.locks != 0 || thisg.m.mallocing != 0 || thisg.m.preemptoff != "" || thisg.m.p.ptr().status != _Prunning {
            // 先不抢占了，之后通过 go.preempt 进行抢占
            // Let the goroutine keep running for now.
            // gp->preempt is set, so it will be preempted next time.
            gp.stackguard0 = gp.stack.lo + _StackGuard
            gogo(&gp.sched) // never return
        }
    }

    if gp.stack.lo == 0 {
        throw("missing stack in newstack")
    }
    sp := gp.sched.sp
    if sys.ArchFamily == sys.AMD64 || sys.ArchFamily == sys.I386 || sys.ArchFamily == sys.WASM {
        // The call to morestack cost a word.
        sp -= sys.PtrSize
    }
    if stackDebug >= 1 || sp < gp.stack.lo {
        print("runtime: newstack sp=", hex(sp), " stack=[", hex(gp.stack.lo), ", ", hex(gp.stack.hi), "]\n",
            "\tmorebuf={pc:", hex(morebuf.pc), " sp:", hex(morebuf.sp), " lr:", hex(morebuf.lr), "}\n",
            "\tsched={pc:", hex(gp.sched.pc), " sp:", hex(gp.sched.sp), " lr:", hex(gp.sched.lr), " ctxt:", gp.sched.ctxt, "}\n")
    }
    if sp < gp.stack.lo {
        print("runtime: gp=", gp, ", goid=", gp.goid, ", gp->status=", hex(readgstatus(gp)), "\n ")
        print("runtime: split stack overflow: ", hex(sp), " < ", hex(gp.stack.lo), "\n")
        throw("runtime: split stack overflow")
    }

    if preempt {
        if gp == thisg.m.g0 {
            throw("runtime: preempt g0")
        }
        if thisg.m.p == 0 && thisg.m.locks == 0 {
            throw("runtime: g is running but p is not")
        }
        // Synchronize with scang.
        casgstatus(gp, _Grunning, _Gwaiting)
        // gc 抢占优先
        if gp.preemptscan {
            for !castogscanstatus(gp, _Gwaiting, _Gscanwaiting) {
                // Likely to be racing with the GC as
                // it sees a _Gwaiting and does the
                // stack scan. If so, gcworkdone will
                // be set and gcphasework will simply
                // return.
            }
            if !gp.gcscandone {
                // gcw is safe because we're on the
                // system stack.
                gcw := &gp.m.p.ptr().gcw
                scanstack(gp, gcw)
                gp.gcscandone = true
            }
            gp.preemptscan = false
            gp.preempt = false
            casfrom_Gscanstatus(gp, _Gscanwaiting, _Gwaiting)
            // This clears gcscanvalid.
            casgstatus(gp, _Gwaiting, _Grunning)
            gp.stackguard0 = gp.stack.lo + _StackGuard
            gogo(&gp.sched) // never return
        }

        // Act like goroutine called runtime.Gosched.
        casgstatus(gp, _Gwaiting, _Grunning)
        // 抢占 G，将 gp 放到全局等待队列后，触发调度
        gopreempt_m(gp) // never return
    }

    // Allocate a bigger segment and move the stack.
    oldsize := gp.stack.hi - gp.stack.lo
    newsize := oldsize * 2 // 扩容一倍
    if newsize > maxstacksize { // 64bit 系统 maxstacksize 为 1G
        print("runtime: goroutine stack exceeds ", maxstacksize, "-byte limit\n")
        throw("stack overflow")
    }

    // The goroutine must be executing in order to call newstack,
    // so it must be Grunning (or Gscanrunning).
    casgstatus(gp, _Grunning, _Gcopystack)

    // The concurrent GC will not scan the stack while we are doing the copy since
    // the gp is in a Gcopystack status.
    // 将旧栈的数据复制到新的栈上
    copystack(gp, newsize, true)
    if stackDebug >= 1 {
        print("stack grow done\n")
    }
    casgstatus(gp, _Gcopystack, _Grunning)
    gogo(&gp.sched) // 切到新的栈上继续执行
}

// Copies gp's stack to a new stack of a different size.
// Caller must have changed gp status to Gcopystack.
//
// If sync is true, this is a self-triggered stack growth and, in
// particular, no other G may be writing to gp's stack (e.g., via a
// channel operation). If sync is false, copystack protects against
// concurrent channel operations.
// 如果 sync 参数为 true，表示是当前 g 自己触发的栈扩容，不会有其它 g 写当前 g 的栈（例如 channel 操作）
// 如果 sync 参数为 false（当执行栈缩容的时候），就需要防止并发的 channel 操作了
func copystack(gp *g, newsize uintptr, sync bool) {
    if gp.syscasllsp != 0 {
                throw("stack growth not allowed in system call")
            }
            old := gp.stack
            if old.lo == 0 {
                throw("nil stackbase")
            }
            used := old.hi - gp.sched.sp // 当前已使用的栈空间大小
        
            // allocate new stack
            new := stackalloc(uint32(newsize)) // 分配栈空间，上文中已分析过
            if stackPoisonCopy != 0 {
                fillstack(new, 0xfd)
            }
            if stackDebug >= 1 {
                print("copystack gp=", gp, " [", hex(old.lo), " ", hex(old.hi-used), " ", hex(old.hi), "]", " -> [", hex(new.lo), " ", hex(new.hi-used), " ", hex(new.hi), "]/", newsize, "\n")
            }
        
            // Compute adjustment.
            var adjinfo adjustinfo
            adjinfo.old = old
            adjinfo.delta = new.hi - old.hi
        
            // Adjust sudogs, synchronizing with channel ops if necessary.
            ncopy := used
            if ync {
        adjustsudogs(gp, &adjinfo) // 调整 sudog 中的指针
    } else {
        // sudogs can point in to the stack. During concurrent
        // shrinking, these areas may be written to. Find the
        // highest such pointer so we can handle everything
        // there and below carefully. (This shouldn't be far
        // from the bottom of the stack, so there's little
        // cost in handling everything below it carefully.)
        adjinfo.sghi = findsghi(gp, old) // 在所有 sudog 中找到地址最大的指针

        // Synchronize with channel ops and copy the part of
        // the stack they may interact with.
        // 对所有 sudog 关联的 channel 上锁，然后调整指针，并且复制 sudog 指向的部分旧栈的数据到新的栈上
        ncopy -= syncadjustsudogs(gp, used, &adjinfo)
    }

    // Copy the stack (or the rest of it) to the new location
    memmove(unsafe.Pointer(new.hi-ncopy), unsafe.Pointer(old.hi-ncopy), ncopy)

    // Adjust remaining structures that have pointers into stacks.
    // We have to do most of these before we traceback the new
    // stack because gentraceback uses them.
    adjustctxt(gp, &adjinfo)
    adjustdefers(gp, &adjinfo)
    adjustpanics(gp, &adjinfo)
    if adjinfo.sghi != 0 {
        adjinfo.sghi += adjinfo.delta
    }

    // Swap out old stack for new one
    gp.stack = new
    gp.stackguard0 = new.lo + _StackGuard // NOTE: might clobber a preempt request
    gp.sched.sp = new.hi - used
    gp.stktopsp += adjinfo.delta

    // Adjust pointers in the new stack.
    gentraceback(^uintptr(0), ^uintptr(0), 0, gp, 0, nil, 0x7fffffff, adjustframe, noescape(unsafe.Pointer(&adjinfo)), 0)

    // free old stack
    if stackPoisonCopy != 0 {
        fillstack(old, 0xfc)
    }
    stackfree(old)
}
```

从新的栈上继续执行当前 g：

```text
// func gogo(buf *gobuf)
// restore state from Gobuf; longjmp
TEXT runtime·gogo(SB), NOSPLIT, $16-8
    MOVQ    buf+0(FP), BX        // gobuf
    MOVQ    gobuf_g(BX), DX
    MOVQ    0(DX), CX        // make sure g != nil
    get_tls(CX)
    MOVQ    DX, g(CX)
    MOVQ    gobuf_sp(BX), SP    // restore SP
    MOVQ    gobuf_ret(BX), AX
    MOVQ    gobuf_ctxt(BX), DX
    MOVQ    gobuf_bp(BX), BP
    MOVQ    $0, gobuf_sp(BX)    // clear to help garbage collector
    MOVQ    $0, gobuf_ret(BX)
    MOVQ    $0, gobuf_ctxt(BX)
    MOVQ    $0, gobuf_bp(BX)
    MOVQ    gobuf_pc(BX), BX
    JMP    BX
```

#### 栈的缩容

栈的缩容是发生在垃圾回收时，主动触发的。在`runtime.scanstack`函数中调用，如果当前使用的栈空间小于可用栈空间的1/4，则执行缩容：

```go
// Maybe shrink the stack being used by gp.
// Called at garbage collection time.
// gp must be stopped, but the world need not be.
func shrinkstack(gp *g) {
    gstatus := readgstatus(gp)
    if gp.stack.lo == 0 {
        throw("missing stack in shrinkstack")
    }
    if gstatus&_Gscan == 0 {
        throw("bad status in shrinkstack")
    }

    if debug.gcshrinkstackoff > 0 {
        return
    }
    f := findfunc(gp.startpc)
    if f.valid() && f.funcID == funcID_gcBgMarkWorker {
        // We're not allowed to shrink the gcBgMarkWorker
        // stack (see gcBgMarkWorker for explanation).
        return
    }

    oldsize := gp.stack.hi - gp.stack.lo
    newsize := oldsize / 2 // 缩小一倍
    // Don't shrink the allocation below the minimum-sized stack
    // allocation.
    if newsize < _FixedStack {
        return
    }
    // Compute how much of the stack is currently in use and only
    // shrink the stack if gp is using less than a quarter of its
    // current stack. The currently used stack includes everything
    // down to the SP plus the stack guard space that ensures
    // there's room for nosplit functions.
    avail := gp.stack.hi - gp.stack.lo
    // 如果当前使用的栈空间已经达到可用栈空间的1/4，则不进行缩容
    if used := gp.stack.hi - gp.sched.sp + _StackLimit; used >= avail/4 {
        return
    }

    // We can't copy the stack if we're in a syscall.
    // The syscall might have pointers into the stack.
    if gp.syscallsp != 0 {
        return
    }
    if sys.GoosWindows != 0 && gp.m != nil && gp.m.libcallsp != 0 {
        return
    }

    if stackDebug > 0 {
        print("shrinking stack ", oldsize, "->", newsize, "\n")
    }

    copystack(gp, newsize, false)
}
```

#### 栈的回收

栈的回收类似于栈的分配的逆过程。

```go
// stackfree frees an n byte stack allocation at stk.
//
// stackfree must run on the system stack because it uses per-P
// resources and must not split the stack.
//
//go:systemstack
func stackfree(stk stack) {
	gp := getg()
	v := unsafe.Pointer(stk.lo)
	n := stk.hi - stk.lo
	if n&(n-1) != 0 {
		throw("stack not a power of 2")
	}
	if stk.lo+n < stk.hi {
		throw("bad stack size")
	}
	if stackDebug >= 1 {
		println("stackfree", v, n)
		memclrNoHeapPointers(v, n) // for testing, clobber stack data
	}
	if debug.efence != 0 || stackFromSystem != 0 {
		if debug.efence != 0 || stackFaultOnFree != 0 {
			sysFault(v, n)
		} else {
			sysFree(v, n, &memstats.stacks_sys)
		}
		return
	}
	if msanenabled {
		msanfree(v, n)
	}
    // 栈空间比较小，放回本地缓存中，或者全局的栈空间池子中
	if n < _FixedStack<<_NumStackOrders && n < _StackCacheSize {
		order := uint8(0)
		n2 := n
		for n2 > _FixedStack {
			order++
			n2 >>= 1
		}
		x := gclinkptr(v)
		c := gp.m.mcache
		if stackNoCache != 0 || c == nil || gp.m.preemptoff != "" {
			lock(&stackpoolmu)
			stackpoolfree(x, order)
			unlock(&stackpoolmu)
		} else {
			if c.stackcache[order].size >= _StackCacheSize {
				stackcacherelease(c, order)
			}
			x.ptr().next = c.stackcache[order].list
			c.stackcache[order].list = x
			c.stackcache[order].size += n
		}
	} else {
        // 大栈空间
        // 如果正在进行 GC，放回全局的大栈空间池子中
        // 否则，由 mheap 回收
		s := spanOfUnchecked(uintptr(v))
		if s.state != mSpanManual {
			println(hex(s.base()), v)
			throw("bad span state")
		}
		if gcphase == _GCoff {
			// Free the stack immediately if we're
			// sweeping.
			osStackFree(s)
			mheap_.freeManual(s, &memstats.stacks_inuse)
		} else {
			// If the GC is running, we can't return a
			// stack span to the heap because it could be
			// reused as a heap span, and this state
			// change would race with GC. Add it to the
			// large stack cache instead.
			log2npage := stacklog2(s.npages)
			lock(&stackLarge.lock)
			stackLarge.free[log2npage].insert(s)
			unlock(&stackLarge.lock)
		}
	}
}
```