---
layout: post
title: "golang channel底层实现"
description: "golang channel底层实现"
category: GoLang
tags: []
---
{% include JB/setup %}

本文内容基于go1.13.1源码。

从 golang 官方文档[https://golang.org/ref/spec#Channel_types](https://golang.org/ref/spec#Channel_types)可以了解到 channel 数据类型所支持的操作，接下来我将尝试探索这些操作底层是如何实现的：

1. channel 底层数据结构的实现？
1. channel 是如何创建、发送、接收的？
1. 只能发送（chan<- int)）和只能接收（<-chan int）的 channel 是如何实现的？
1. select 是怎么实现同时监听多个 channel 的？

<!--more-->

测试用的代码：

```go
package main

func main() {
    ch := make(chan string, 1)
    ch <- "hello"
    _ = <-ch
}
```

通过`go tool compile  -N -l -S channel.go`，可以知道 channel 的相关操作分别调用了：
* 创建：runtime.makechan，其中`make(chan string)`，实际上等于`make(chan string, 0)`
* 发送：runtime.chansend1
* 接收：runtime.chanrecv1

channel 的数据结构：

```go
type hchan struct {
    qcount   uint           // total data in the queue
    dataqsiz uint           // 缓冲队列大小，0表示无缓冲的channel
    buf      unsafe.Pointer // points to an array of dataqsiz elements
    elemsize uint16
    closed   uint32 // 1 表示 close(channel)，0 表示未 close
    elemtype *_type // element type
    sendx    uint   // send index 指向缓冲队列的尾部
    recvx    uint   // receive index 指向缓冲队列的头部
    recvq    waitq  // list of recv waiters 等待接收数据的 g
    sendq    waitq  // list of send waiters 等待发送数据的 g

    // lock protects all fields in hchan, as well as several
    // fields in sudogs blocked on this channel.
    //
    // Do not change another G's status while holding this lock
    // (in particular, do not ready a G), as this can deadlock
    // with stack shrinking.
    lock mutex
}

// 等待队列
type waitq struct {
    first *sudog
    last  *sudog
}
```

#### 创建

```go
func makechan(t *chantype, size int) *hchan {
    elem := t.elem

    // compiler checks this but be safe.
    if elem.size >= 1<<16 {
        throw("makechan: invalid channel element type")
    }
    if hchanSize%maxAlign != 0 || elem.align > maxAlign {
        throw("makechan: bad alignment")
    }

    mem, overflow := math.MulUintptr(elem.size, uintptr(size))
    if overflow || mem > maxAlloc-hchanSize || size < 0 {
        panic(plainError("makechan: size out of range"))
    }

    // Hchan does not contain pointers interesting for GC when elements stored in buf do not contain pointers.
    // buf points into the same allocation, elemtype is persistent.
    // SudoG's are referenced from their owning thread so they can't be collected.
    // TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
    var c *hchan
    switch {
    case mem == 0:
        // 无缓冲 channel
        // Queue or element size is zero.
        c = (*hchan)(mallocgc(hchanSize, nil, true))
        // Race detector uses this location for synchronization.
        c.buf = c.raceaddr()
    case elem.ptrdata == 0:
        // channel 元素类型不包含指针，将 hchan 和 buf 的内存分配在一起
        // Elements do not contain pointers.
        // Allocate hchan and buf in one call.
        c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
        c.buf = add(unsafe.Pointer(c), hchanSize)
    default:
        // channel 元素类型包含指针，hchan 和 buf 的内存分开分配
        // Elements contain pointers.
        c = new(hchan)
        c.buf = mallocgc(mem, elem, true)
    }

    c.elemsize = uint16(elem.size)
    c.elemtype = elem
    c.dataqsiz = uint(size) // 缓冲区大小

    if debugChan {
        print("makechan: chan=", c, "; elemsize=", elem.size, "; elemalg=", elem.alg, "; dataqsiz=", size, "\n")
    }
    return c
}
```

#### 发送

```go
// 阻塞发送
func chansend1(c *hchan, elem unsafe.Pointer) {
    chansend(c, elem, true, getcallerpc())
}

// 参数 block 表示发送操作是否阻塞
// 参数 ep 是发送数据的地址
// 返回结果表示 是否发送成功
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    if c == nil {
        if !block {
            return false
        }
        // 从 nil 值的 channel 变量接收数据会永远阻塞
        gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }

    if debugChan {
        print("chansend: chan=", c, "\n")
    }

    if raceenabled {
        racereadpc(c.raceaddr(), callerpc, funcPC(chansend))
    }

    // 下面这段注释主要解释了为何不进行加锁就可以判断非阻塞操作是否可以立即返回
    // 首先对于对齐的内存单字（32位机器是4个字节，64位机器是8个字节）的并发读写是安全的，这里的安全指的是不会读/写脏数据，即读不会读到别的线程写了一半的数据，写不会出现数据是多个线程各写了一部分，写只会出现覆盖的情况
    // 在下面这个 if 判断中，c.closed，c.recvq.first 和 c.qcount 是有可能被并发读写的（实际上是多读一写，写必须加锁），但是对于返回结果以及程序逻辑并没有影响
    // Fast path: check for failed non-blocking operation without acquiring the lock.
    //
    // After observing that the channel is not closed, we observe that the channel is
    // not ready for sending. Each of these observations is a single word-sized read
    // (first c.closed and second c.recvq.first or c.qcount depending on kind of channel).
    // Because a closed channel cannot transition from 'ready for sending' to
    // 'not ready for sending', even if the channel is closed between the two observations,
    // they imply a moment between the two when the channel was both not yet closed
    // and not ready for sending. We behave as if we observed the channel at that moment,
    // and report that the send cannot proceed.
    //
    // It is okay if the reads are reordered here: if we observe that the channel is not
    // ready for sending and then observe that it is not closed, that implies that the
    // channel wasn't closed during the first observation.
    if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) ||
        (c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
        return false
    }

    var t0 int64
    if blockprofilerate > 0 {
        t0 = cputicks()
    }

    lock(&c.lock)

    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("send on closed channel"))
    }

    // 如果有接收的 g 在等待，说明 buffer 里没有数据，直接发给等待的 g
    if sg := c.recvq.dequeue(); sg != nil {
        // Found a waiting receiver. We pass the value we want to send
        // directly to the receiver, bypassing the channel buffer (if any).
        send(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true
    }

    // 如果 buffer 还有空闲的位置
    if c.qcount < c.dataqsiz {
        // Space is available in the channel buffer. Enqueue the element to send.
        qp := chanbuf(c, c.sendx)
        if raceenabled {
            raceacquire(qp)
            racerelease(qp)
        }
        typedmemmove(c.elemtype, qp, ep)
        c.sendx++
        if c.sendx == c.dataqsiz {
            c.sendx = 0
        }
        c.qcount++
        unlock(&c.lock)
        return true
    }

    // buffer 没有空闲的位置，该操作是非阻塞的，立即返回
    if !block {
        unlock(&c.lock)
        return false
    }

    // 阻塞当前 g，把当前 g 入到发送队列
    // Block on the channel. Some receiver will complete our operation for us.
    // 获取当前的 g
    gp := getg()
    mysg := acquireSudog()
    mysg.releasetime = 0
    if t0 != 0 {
        mysg.releasetime = -1
    }
    // No stack splits between assigning elem and enqueuing mysg
    // on gp.waiting where copystack can find it.
    mysg.elem = ep // 发送数据的地址
    mysg.waitlink = nil
    mysg.g = gp
    mysg.isSelect = false // 是否来自 select{}
    mysg.c = c
    gp.waiting = mysg
    gp.param = nil
    // 入队
    c.sendq.enqueue(mysg)
    // 将当前 g 挂起，并且释放 c.lock，为何释放 c.lock 要在 goparkunlock 里面的调用做？
    // 下面的 2 个函数调用跟 g 的实现相关，先不深究
    // 在这里阻塞的 g 会在 chan.go:554 recv() 函数中唤醒
    goparkunlock(&c.lock, waitReasonChanSend, traceEvGoBlockSend, 3)
    // Ensure the value being sent is kept alive until the
    // receiver copies it out. The sudog has a pointer to the
    // stack object, but sudogs aren't considered as roots of the
    // stack tracer.
    KeepAlive(ep)

    // 挂起的 g 被唤醒，继续执行
    // someone woke us up.
    if mysg != gp.waiting {
        throw("G waiting list is corrupted")
    }
    gp.waiting = nil
    if gp.param == nil { // 正常唤醒的情况下，gp.param = unsafe.Pointer(mysg)
        if c.closed == 0 { // 异常唤醒，但是 channel 并没有关闭，这种情况不应该出现，如果出现应该是 go 的实现有 bug，所以使用 throw 抛出错误
            throw("chansend: spurious wakeup")
        }
        // channel 被 close 了，所以当有 g 在往 channel 发数据时，不能直接 close，否则就会出现这个错误
        // 因为这是 go 用户代码引起的，所以采用 panic 抛出错误
        panic(plainError("send on closed channel"))
    }
    gp.param = nil
    if mysg.releasetime > 0 {
        blockevent(mysg.releasetime-t0, 2)
    }
    mysg.c = nil
    releaseSudog(mysg)
    return true
}
```

#### 接收

```go
func chanrecv1(c *hchan, elem unsafe.Pointer) {
    chanrecv(c, elem, true)
}

// 参数 ep 是接收数据的地址
// 返回值：
// selected 用于 select{}，表示是否执行对应的 case ... <- channel，在下文中的 selectnbrecv() 用到
// received 用于判断是否接收到了数据
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    // raceenabled: don't need to check ep, as it is always on the stack
    // or is new memory allocated by reflect.

    if debugChan {
        print("chanrecv: chan=", c, "\n")
    }

    if c == nil {
        if !block {
            return
        }
        // 从值为 nil 的 channel 中接收数据，当前 g 将会一直阻塞
        gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
        throw("unreachable")
    }

    // 下面的 if 判断跟 chansend 里的类似，但这里为何需要使用 atomic 包？
    // Fast path: check for failed non-blocking operation without acquiring the lock.
    //
    // After observing that the channel is not ready for receiving, we observe that the
    // channel is not closed. Each of these observations is a single word-sized read
    // (first c.sendq.first or c.qcount, and second c.closed).
    // Because a channel cannot be reopened, the later observation of the channel
    // being not closed implies that it was also not closed at the moment of the
    // first observation. We behave as if we observed the channel at that moment
    // and report that the receive cannot proceed.
    //
    // The order of operations is important here: reversing the operations can lead to
    // incorrect behavior when racing with a close.
    if !block && (c.dataqsiz == 0 && c.sendq.first == nil ||
        c.dataqsiz > 0 && atomic.Loaduint(&c.qcount) == 0) &&
        atomic.Load(&c.closed) == 0 {
        return
    }

    var t0 int64
    if blockprofilerate > 0 {
        t0 = cputicks()
    }

    lock(&c.lock)

    // channel 已 close，且没有数据了
    if c.closed != 0 && c.qcount == 0 {
        if raceenabled {
            raceacquire(c.raceaddr())
        }
        unlock(&c.lock)
        if ep != nil {
            typedmemclr(c.elemtype, ep)
        }
        return true, false
    }

    // 从发送等待队列中取出一个 g
    if sg := c.sendq.dequeue(); sg != nil {
        // 如果是无缓冲 channel，则直接接收来自 sg 的数据
        // 如果是有缓冲 channel，此时缓冲队列一定是满的，从队列头部出队一个数据给当前 g，然后把 sg 发送的数据入队
        // Found a waiting sender. If buffer is size 0, receive value
        // directly from sender. Otherwise, receive from head of queue
        // and add sender's value to the tail of the queue (both map to
        // the same buffer slot because the queue is full).
        recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true, true
    }

    // 如果发送等待队列是空的，从 buffer 中取出数据
    if c.qcount > 0 {
        // Receive directly from queue
        qp := chanbuf(c, c.recvx)
        if raceenabled {
            raceacquire(qp)
            racerelease(qp)
        }
        if ep != nil {
            typedmemmove(c.elemtype, ep, qp)
        }
        typedmemclr(c.elemtype, qp)
        c.recvx++ // 缓冲队列的头部
        if c.recvx == c.dataqsiz {
            c.recvx = 0
        }
        c.qcount--
        unlock(&c.lock)
        return true, true
    }

    // channel 没有数据，且非阻塞，直接返回
    if !block {
        unlock(&c.lock)
        return false, false
    }

    // 阻塞等待数据
    // no sender available: block on this channel.
    gp := getg()
    mysg := acquireSudog()
    mysg.releasetime = 0
    if t0 != 0 {
        mysg.releasetime = -1
    }
    // No stack splits between assigning elem and enqueuing mysg
    // on gp.waiting where copystack can find it.
    mysg.elem = ep // 接收数据的地址，会在 chan.go:269 send() 函数中用到
    mysg.waitlink = nil
    gp.waiting = mysg
    mysg.g = gp
    mysg.isSelect = false
    mysg.c = c
    gp.param = nil
    c.recvq.enqueue(mysg)
    // 挂起当前 g，会在 chan.go:269 send() 函数中被唤醒
    goparkunlock(&c.lock, waitReasonChanReceive, traceEvGoBlockRecv, 3)

    // someone woke us up
    if mysg != gp.waiting {
        throw("G waiting list is corrupted")
    }
    gp.waiting = nil
    if mysg.releasetime > 0 {
        blockevent(mysg.releasetime-t0, 2)
    }
    closed := gp.param == nil
    gp.param = nil
    mysg.c = nil
    releaseSudog(mysg)
    return true, !closed
}
```

#### 发送和接收图示

创建2个 buffer 元素为 string 类型的 channel，并且使用一个 goroutine 往 channel 发送3个数据。

![](/assets/img/201910250101.jpeg)

有一个 goroutine 从 channel 中接收1个数据：

![](/assets/img/201910260101.jpeg)

有一个 goroutine 从 channel 中接收2个数据：

![](/assets/img/201910260102.jpeg)

#### 只能发送/接收的 channel

当往`<-chan string`发送数据会遇到错误：`invalid operation: ch <- fmt.Sprintf("block%d", i) (send to receive-only type <-chan string)`

当从`chan<- string`接收数据会遇到错误：`invalid operation: <-ch (receive from send-only type chan<- string)`

在 go 源码搜索错误消息可以发现上面的报错属于语法错误。

#### select 监听多个 channel 的实现

参考文章：[浅谈 Go 语言实现原理 4.2 select](https://draveness.me/golang/keyword/golang-select.html)

select 根据 case 情况的不同，会生成不同的程序逻辑（由编译器在生成中间代码时处理的）。分如下四种情况：

1. select 中没有 case：将会一直阻塞
1. select 中只存在一个 case（default 可以认为是一种特殊的 case）：
    1. 如果是 default：则直接执行 default
    1. 如果不是 default：则跟直接使用 channel 等价
1. select 中有两个 case，其中一个是 default，将会优化为 if/else 语句：
    1. 如果是发送操作：`if runtime.selectnbsend(ch, elem) { /* case ch <- elem 代码块 */ } else { /* default 代码块 */ }`
    1. 如果是接收操作：`if runtime.selectnbrecv(elem, ch) { /* case elem <- ch 代码块 */ } else { /* default 代码块 */ }`
1. select 中有 >=2个非 default 的 case，将会转换为对 `runtime.selectgo` 的调用，该调用会返回要执行的 case 代码块索引

runtime.selectnbrecv 的实现：

```go
// compiler implements
//
//    select {
//    case v = <-c:
//        ... foo
//    default:
//        ... bar
//    }
//
// as
//
//    if selectnbrecv(&v, c) {
//        ... foo
//    } else {
//        ... bar
//    }
//
func selectnbrecv(elem unsafe.Pointer, c *hchan) (selected bool) {
    selected, _ = chanrecv(c, elem, false) // 实际调用的 chanrecv 上文已经分析过了
    return
}
```

runtime.selectgo 的实现：

```go
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
    if debugSelect {
        print("select: cas0=", cas0, "\n")
    }

    cas1 := (*[1 << 16]scase)(unsafe.Pointer(cas0))
    order1 := (*[1 << 17]uint16)(unsafe.Pointer(order0))

    // case 条件（包括default）
    scases := cas1[:ncases:ncases]
    // 用于实现当有多个 channel 都 ready 的时候，随机选择一个 case
    pollorder := order1[:ncases:ncases]
    // 对所有 case 的 channel 进行加锁操作的顺序
    lockorder := order1[ncases:][:ncases:ncases]

    // Replace send/receive cases involving nil channels with
    // caseNil so logic below can assume non-nil channel.
    for i := range scases {
        cas := &scases[i]
        if cas.c == nil && cas.kind != caseDefault {
            *cas = scase{}
        }
    }

    var t0 int64
    if blockprofilerate > 0 {
        t0 = cputicks()
        for i := 0; i < ncases; i++ {
            scases[i].releasetime = -1
        }
    }

    // The compiler rewrites selects that statically have
    // only 0 or 1 cases plus default into simpler constructs.
    // The only way we can end up with such small sel.ncase
    // values here is for a larger select in which most channels
    // have been nilled out. The general code handles those
    // cases correctly, and they are rare enough not to bother
    // optimizing (and needing to test).

    // generate permuted order
    for i := 1; i < ncases; i++ {
        j := fastrandn(uint32(i + 1)) // 返回 [0, i] 的随机值
        pollorder[i] = pollorder[j]
        pollorder[j] = uint16(i)
    }
    // 当 ncases = 3 时，可能会产生如下的顺序：
    // pollorder = []uint16 len: 3, cap: 3, [1,2,0]

    // 按照 channel 的内存地址进行排序，得到上锁的顺序，关于为何需要计算上锁顺序会在下文用一个例子讲解
    // sort the cases by Hchan address to get the locking order.
    // simple heap sort, to guarantee n log n time and constant stack footprint.
    // 堆排序，先建立大顶堆
    for i := 0; i < ncases; i++ {
        j := i
        // Start with the pollorder to permute cases on the same channel.
        c := scases[pollorder[i]].c
        for j > 0 && scases[lockorder[(j-1)/2]].c.sortkey() < c.sortkey() {
            k := (j - 1) / 2
            lockorder[j] = lockorder[k]
            j = k
        }
        lockorder[j] = pollorder[i]
    }
    // 堆排序，对大顶堆进行调整
    for i := ncases - 1; i >= 0; i-- {
        o := lockorder[i]
        c := scases[o].c
        lockorder[i] = lockorder[0]
        j := 0
        for {
            k := j*2 + 1
            if k >= i {
                break
            }
            if k+1 < i && scases[lockorder[k]].c.sortkey() < scases[lockorder[k+1]].c.sortkey() {
                k++
            }
            if c.sortkey() < scases[lockorder[k]].c.sortkey() {
                lockorder[j] = lockorder[k]
                j = k
                continue
            }
            break
        }
        lockorder[j] = o
    }

    if debugSelect {
        for i := 0; i+1 < ncases; i++ {
            if scases[lockorder[i]].c.sortkey() > scases[lockorder[i+1]].c.sortkey() {
                print("i=", i, " x=", lockorder[i], " y=", lockorder[i+1], "\n")
                throw("select: broken sort")
            }
        }
    }

    // 根据 lockorder 的顺序，对 channel 进行上锁，相同 channel 不会重复上锁
    // lock all the channels involved in the select
    sellock(scases, lockorder)

    var (
        gp     *g
        sg     *sudog
        c      *hchan
        k      *scase
        sglist *sudog
        sgnext *sudog
        qp     unsafe.Pointer
        nextp  **sudog
    )

loop:
    // pass 1 - look for something already waiting
    var dfli int
    var dfl *scase
    var casi int
    var cas *scase
    var recvOK bool
    for i := 0; i < ncases; i++ {
        casi = int(pollorder[i])
        cas = &scases[casi]
        c = cas.c

        switch cas.kind {
        case caseNil:
            continue

        case caseRecv: // 关闭的 channel 是可以执行 recv 操作的，所以先判断是否有数据
            sg = c.sendq.dequeue()
            if sg != nil { // channel 发送队列不为空，肯定能接收到数据
                goto recv
            }
            if c.qcount > 0 { // channel 缓冲队列不为空，也能接收到数据
                goto bufrecv
            }
            if c.closed != 0 { // channel 关闭了
                goto rclose
            }

        case caseSend: // 关闭的 channel 是不可以执行 send 操作的，所以先判断是否关闭
            if raceenabled {
                racereadpc(c.raceaddr(), cas.pc, chansendpc)
            }
            if c.closed != 0 { // channel 关闭了
                goto sclose
            }
            sg = c.recvq.dequeue()
            if sg != nil { // channel 接收队列不为空，肯定能发送
                goto send
            }
            if c.qcount < c.dataqsiz { // channel 缓冲队列未满，也能发送
                goto bufsend
            }

        case caseDefault:
            dfli = casi
            dfl = cas
        }
    }

    // 没有 channel 是 ready 的，执行 default 语句
    if dfl != nil {
        selunlock(scases, lockorder)
        casi = dfli
        cas = dfl
        goto retc
    }

    // 没有 channel 是 ready 的，且没有 default 语句
    // 将当前 g 挂到所有 channel 等待队列上
    // pass 2 - enqueue on all chans
    gp = getg()
    if gp.waiting != nil {
        throw("gp.waiting != nil")
    }
    nextp = &gp.waiting
    for _, casei := range lockorder {
        casi = int(casei)
        cas = &scases[casi]
        if cas.kind == caseNil {
            continue
        }
        c = cas.c
        sg := acquireSudog()
        sg.g = gp
        sg.isSelect = true // 标记是从 select 入队的
        // No stack splits between assigning elem and enqueuing
        // sg on gp.waiting where copystack can find it.
        sg.elem = cas.elem
        sg.releasetime = 0
        if t0 != 0 {
            sg.releasetime = -1
        }
        sg.c = c
        // Construct waiting list in lock order.
        *nextp = sg
        nextp = &sg.waitlink

        switch cas.kind {
        case caseRecv:
            c.recvq.enqueue(sg)

        case caseSend:
            c.sendq.enqueue(sg)
        }
    }

    // wait for someone to wake us up
    gp.param = nil
    // 在 selparkcommit 中会通过 g.p.waiting 释放所有 channel 的锁
    // 当有多个 channel ready时，多个 channel 的等待队列在将 g 出队时，会判断该 g 是否通过 select 入队的，如果是，检查 spg.g.selectDone 是否标记为1了，如果是就会跳过，否则就唤醒，对应的代码是 runtime/chan.go:732 func (q *waitq) dequeue() *sudog {}
    gopark(selparkcommit, nil, waitReasonSelect, traceEvGoBlockSelect, 1)

    // 有 channel ready 了，重新对所有 channel 上锁
    sellock(scases, lockorder)

    gp.selectDone = 0
    sg = (*sudog)(gp.param)
    gp.param = nil

    // pass 3 - dequeue from unsuccessful chans
    // otherwise they stack up on quiet channels
    // record the successful case, if any.
    // We singly-linked up the SudoGs in lock order.
    casi = -1
    cas = nil
    sglist = gp.waiting
    // Clear all elem before unlinking from gp.waiting.
    for sg1 := gp.waiting; sg1 != nil; sg1 = sg1.waitlink {
        sg1.isSelect = false
        sg1.elem = nil
        sg1.c = nil
    }
    gp.waiting = nil

    for _, casei := range lockorder {
        k = &scases[casei]
        if k.kind == caseNil {
            continue
        }
        if sglist.releasetime > 0 {
            k.releasetime = sglist.releasetime
        }
        // 找到唤醒当前 g 的是哪个 channel
        if sg == sglist {
            // sg has already been dequeued by the G that woke us up.
            casi = int(casei)
            cas = k
        } else {
            // 将当前 g 从其他 channel 的等待队列中出队
            c = k.c
            if k.kind == caseSend {
                c.sendq.dequeueSudoG(sglist)
            } else {
                c.recvq.dequeueSudoG(sglist)
            }
        }
        sgnext = sglist.waitlink
        sglist.waitlink = nil
        releaseSudog(sglist)
        sglist = sgnext
    }

    // 某个 channel 被 close 导致的唤醒
    // 跳转到 loop，判断是接收 channel 被 close，还是发送 channel 被 close
    if cas == nil {
        // We can wake up with gp.param == nil (so cas == nil)
        // when a channel involved in the select has been closed.
        // It is easiest to loop and re-run the operation;
        // we'll see that it's now closed.
        // Maybe some day we can signal the close explicitly,
        // but we'd have to distinguish close-on-reader from close-on-writer.
        // It's easiest not to duplicate the code and just recheck above.
        // We know that something closed, and things never un-close,
        // so we won't block again.
        goto loop
    }

    c = cas.c

    if debugSelect {
        print("wait-return: cas0=", cas0, " c=", c, " cas=", cas, " kind=", cas.kind, "\n")
    }

    if cas.kind == caseRecv {
        recvOK = true
    }

    if raceenabled {
        if cas.kind == caseRecv && cas.elem != nil {
            raceWriteObjectPC(c.elemtype, cas.elem, cas.pc, chanrecvpc)
        } else if cas.kind == caseSend {
            raceReadObjectPC(c.elemtype, cas.elem, cas.pc, chansendpc)
        }
    }
    if msanenabled {
        if cas.kind == caseRecv && cas.elem != nil {
            msanwrite(cas.elem, c.elemtype.size)
        } else if cas.kind == caseSend {
            msanread(cas.elem, c.elemtype.size)
        }
    }

    selunlock(scases, lockorder)
    goto retc

bufrecv:
    // can receive from buffer
    if raceenabled {
        if cas.elem != nil {
            raceWriteObjectPC(c.elemtype, cas.elem, cas.pc, chanrecvpc)
        }
        raceacquire(chanbuf(c, c.recvx))
        racerelease(chanbuf(c, c.recvx))
    }
    if msanenabled && cas.elem != nil {
        msanwrite(cas.elem, c.elemtype.size)
    }
    recvOK = true
    qp = chanbuf(c, c.recvx)
    if cas.elem != nil {
        typedmemmove(c.elemtype, cas.elem, qp)
    }
    typedmemclr(c.elemtype, qp)
    c.recvx++
    if c.recvx == c.dataqsiz {
        c.recvx = 0
    }
    c.qcount--
    selunlock(scases, lockorder)
    goto retc

bufsend:
    // can send to buffer
    if raceenabled {
        raceacquire(chanbuf(c, c.sendx))
        racerelease(chanbuf(c, c.sendx))
        raceReadObjectPC(c.elemtype, cas.elem, cas.pc, chansendpc)
    }
    if msanenabled {
        msanread(cas.elem, c.elemtype.size)
    }
    typedmemmove(c.elemtype, chanbuf(c, c.sendx), cas.elem)
    c.sendx++
    if c.sendx == c.dataqsiz {
        c.sendx = 0
    }
    c.qcount++
    selunlock(scases, lockorder)
    goto retc

recv:
    // can receive from sleeping sender (sg)
    recv(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
    if debugSelect {
        print("syncrecv: cas0=", cas0, " c=", c, "\n")
    }
    recvOK = true
    goto retc

rclose:
    // read at end of closed channel
    selunlock(scases, lockorder)
    recvOK = false
    if cas.elem != nil {
        typedmemclr(c.elemtype, cas.elem)
    }
    if raceenabled {
        raceacquire(c.raceaddr())
    }
    goto retc

send:
    // can send to a sleeping receiver (sg)
    if raceenabled {
        raceReadObjectPC(c.elemtype, cas.elem, cas.pc, chansendpc)
    }
    if msanenabled {
        msanread(cas.elem, c.elemtype.size)
    }
    send(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
    if debugSelect {
        print("syncsend: cas0=", cas0, " c=", c, "\n")
    }
    goto retc

retc:
    if cas.releasetime > 0 {
        blockevent(cas.releasetime-t0, 1)
    }
    return casi, recvOK

sclose:
    // send on closed channel
    selunlock(scases, lockorder)
    panic(plainError("send on closed channel"))
}
```

#### select 实现中对多个 channel 的上锁顺序

考虑如下代码中的2个channel：

```go
package main

func main() {
    ch0 := make(chan string)
    ch1 := make(chan string)

    go func() {
        select {
        case ch0 <- "g1":
            println("g1 send to ch0")
        case ch1 <- "g1":
            println("g1 send to ch1")
        }
    }()
    go func() {
        select {
        case ch1 <- "g2":
            println("g2 send to ch1")
        case ch0 <- "g2":
            println("g2 send to ch0")
        }
    }()
    println(<-ch0)
    println(<-ch1)
}
```

g1 和 g2 中 select 的 case 的 channel 顺序是相反的，如果按照 case 的顺序上锁，很有可能就出现死锁了，所以为了解决死锁问题，就按照 channel 的内存地址排序（由小到大）后，逐个进行加锁。

#### 遗留问题

为何 channel 未就绪时（不能接收/发送）， g 入队后， channel 的锁都是交给 g0 去释放的，而不能在入队之后就立即释放？