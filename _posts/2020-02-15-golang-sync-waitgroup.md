---
layout: post
title: "golang sync.WaitGroup 底层实现"
description: ""
category: 
tags: []
---
{% include JB/setup %}

#### 数据结构

```go
// A WaitGroup waits for a collection of goroutines to finish.
// The main goroutine calls Add to set the number of
// goroutines to wait for. Then each of the goroutines
// runs and calls Done when finished. At the same time,
// Wait can be used to block until all goroutines have finished.
//
// A WaitGroup must not be copied after first use.
type WaitGroup struct {
    noCopy noCopy

    // 64-bit value: high 32 bits are counter, low 32 bits are waiter count.
    // 64-bit atomic operations require 64-bit alignment, but 32-bit
    // compilers do not ensure it. So we allocate 12 bytes and then use
    // the aligned 8 bytes in them as state, and the other 4 as storage
    // for the sema.
    state1 [3]uint32
}
```

* noCopy：用于 go vet 检查 sync.WaitGroup 类型变量是否采用了值传递的方式
    * 如果采用了值传递，go vet 检查会抛出错误：call of foo copies lock value: sync.WaitGroup contains sync.noCopy
    * 因为如果采用值传递，那么 state1 就会被复制一份，而对应的信号量并不会跟着复制，所以值传递后复制出来的是一个不可用的 WaitGroup
* state1：12字节内存
    * 4字节用于 wg.ADD 和 wg.Done 的计数
    * 4字节用于 wg.Wait 的计数
    * 4字节用于信号量的唤醒和等待

![](/assets/img/golang-sync-waitgroup.png)

<!--more-->

#### WaitGroup.state()

根据内存地址计算12字节中，哪8字节用于 wg 的计数，哪4字节用于信号量的唤醒和等待。

```go
// state returns pointers to the state and sema fields stored within wg.state1.
func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
    // 判断地址是否8字节对齐
    if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
        return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
    } else {
        return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
    }
}
```

#### WaitGroup.Add() 和 WaitGroup.Done()

```go
// Done decrements the WaitGroup counter by one.
func (wg *WaitGroup) Done() {
    wg.Add(-1)
}
```

WaitGroup.Done() 实际上是调用的 WaitGroup.Add()，来分析下其源码：

```go
func (wg *WaitGroup) Add(delta int) {
    // statep：用于 wg 计数
    // semap：用于信号量唤醒和等待
    statep, semap := wg.state()
    if race.Enabled {
        _ = *statep // trigger nil deref early
        if delta < 0 {
            // Synchronize decrements with Wait.
            race.ReleaseMerge(unsafe.Pointer(wg))
        }
        race.Disable()
        defer race.Enable()
    }
    // delta 是 int 类型，强制转换为 uint64 类型做加法操作，相当于补码运算
    state := atomic.AddUint64(statep, uint64(delta)<<32)
    // 高 32bit 用于 ADD 和 Done
    v := int32(state >> 32)
    // 低 32bit 用于 Wait，统计等待的 goroutine 数量
    w := uint32(state)
    if race.Enabled && delta > 0 && v == int32(delta) {
        // The first increment must be synchronized with Wait.
        // Need to model this as a read, because there can be
        // several concurrent wg.counter transitions from 0.
        race.Read(unsafe.Pointer(semap))
    }
    // v 不应该小于 0，小于 0 说明使用不当
    if v < 0 {
        panic("sync: negative WaitGroup counter")
    }
    // w != 0 说明已经有 Wait 调用，并且当时观察到 v > 0
    // 如果出现 delta > 0，并且 v == int32(delta)，说明发生过 Done 调用，且该 Done 调用观察到 v == 0，会唤醒所有等待的 goroutine，然而实际上 v > 0，这与 WaitGroup 要实现的功能相悖。
    // 所以确保 Wait 和 Add 不进行并发调用的话，Wait 的语意才能得到保证
    if w != 0 && delta > 0 && v == int32(delta) {
        panic("sync: WaitGroup misuse: Add called concurrently with Wait")
    }
    if v > 0 || w == 0 {
        return
    }
    // v == 0 && w > 0 需要唤醒等待的 goroutine

    // This goroutine has set counter to 0 when waiters > 0.
    // Now there can't be concurrent mutations of state:
    // - Adds must not happen concurrently with Wait,
    // - Wait does not increment waiters if it sees counter == 0.
    // Still do a cheap sanity check to detect WaitGroup misuse.
    // 再检测一次是否出现了 Add 和 Wait 并发调用了
    if *statep != state {
        panic("sync: WaitGroup misuse: Add called concurrently with Wait")
    }
    // 重置等待计数为 0
    // Reset waiters count to 0.
    *statep = 0
    // 唤醒所有等待的 goroutine
    for ; w != 0; w-- {
        runtime_Semrelease(semap, false, 0)
    }
}
```

#### WaitGroup.Wait()

```go
// 阻塞等待，直到 WaitGroup 的计数器为 0
// Wait blocks until the WaitGroup counter is zero.
func (wg *WaitGroup) Wait() {
    statep, semap := wg.state()
    if race.Enabled {
        _ = *statep // trigger nil deref early
        race.Disable()
    }
    for {
        state := atomic.LoadUint64(statep)
        v := int32(state >> 32)
        w := uint32(state)
        // 计数器为 0，不需要等待
        if v == 0 {
            // Counter is 0, no need to wait.
            if race.Enabled {
                race.Enable()
                race.Acquire(unsafe.Pointer(wg))
            }
            return
        }
        // Increment waiters count.
        // CAS 操作，递增等待计数器
        if atomic.CompareAndSwapUint64(statep, state, state+1) {
            if race.Enabled && w == 0 {
                // Wait must be synchronized with the first Add.
                // Need to model this is as a write to race with the read in Add.
                // As a consequence, can do the write only for the first waiter,
                // otherwise concurrent Waits will race with each other.
                race.Write(unsafe.Pointer(semap))
            }
            // 在 semap 上进行等待
            runtime_Semacquire(semap)
            // 被唤醒后计数器应该为 0
            if *statep != 0 {
                panic("sync: WaitGroup is reused before previous Wait has returned")
            }
            if race.Enabled {
                race.Enable()
                race.Acquire(unsafe.Pointer(wg))
            }
            return
        }
    }
}
```

#### runtime_Semrelease 和 runtime_Semacquire

信号量唤醒和等待的实现，代码在 src/runtime/sema.go。semap 会被用作信号量计数器，也会被用来索引 semtable，获取 semaRoot。

semaRoot 是平衡二叉树的实现，存储着在不同地址上进行等待的 goroutine 节点，同一地址上进行等待的 goroutine 会以双向链表的形式组织起来，如下图：

![](/assets/img/golang-semaroot.png)

在 runtime_Semacquire 中，如果能递减 semap，则直接返回，否则调用 semaRoot.queue 方法入队，然后调用 goparkunlock 挂起，等待被唤醒。

在 runtime_Semrelease 中，递增 semap，调用 semaRoot.dequeue 方法出队，然后调用 goready 唤醒对应的 goroutine。

![](/assets/img/golang-semrelease-semacquire.png)
