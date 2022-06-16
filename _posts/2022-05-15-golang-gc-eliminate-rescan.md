---
layout: post
title: "Go GC 如何优化掉重新扫描协程栈"
description: "Go GC 如何优化掉重新扫描协程栈"
category: GoLang
tags: []
---
{% include JB/setup %}

[三色垃圾回收算法维基百科](https://en.wikipedia.org/wiki/Tracing_garbage_collection#Tri-color_marking)图解：

![](/assets/img/Animation_of_tri-color_garbage_collection.gif)

如果三色垃圾回收算法是在程序 STW 的时候执行，那么算法的正确性是能够保证的。但如果三色垃圾回收算法在执行的过程中，程序也在执行（在垃圾回收算法中的术语称为赋值器），赋值器会修改对象之间的引用关系，当出现黑色对象引用了白色对象，同时不存在可以通过其他灰色对象直接/间接访问到该白色对象的路径，由于三色垃圾回收算法不会重新扫描黑色对象，就会出现黑色对象引用的白色对象被回收的情况，也就是本该存活的对象被回收器（垃圾回收代码的执行）错误地当做垃圾回收了。

<!--more-->

为确保三色垃圾回收算法在与赋值器并发执行的情况下的正确性，必须确保导致对象丢失的两个条件不会同时出现：

1. 条件一：赋值器将某一指向白色对象的引用写入黑色对象
2. 条件二：从灰色对象出发，最终到达该白色对象的所有路径都被赋值器破坏

因而赋值器在修改对象的引用关系时只需要满足如下的两个不变式之一，就可以避免导致对象丢失的两个条件同时出现：

* 强三色不变式：不存在从黑色对象指向白色对象的指针
* 弱三色不变式：所有被黑色对象引用的白色对象都处于灰色保护状态（即直接或间接从灰色对象可达）

在 Go 1.8 之前，Go 在实现并发三色垃圾回收算法时，采用的是 Dijkstra 插入写屏障。但出于性能考虑，只在全局变量指针/堆指针上启用写屏障，栈指针不启用。所以在每次并发标记结束时，STW 阶段需要对协程栈进行重新扫描，标记栈对象引用的的堆对象。

Dijkstra 插入屏障伪代码：

```text
writePointer(slot, ptr):
    shade(ptr)
    *slot = ptr
```

**被引用的对象（ptr）如果是白色，则标记为灰色。这确保了不会有黑色对象引用白色对象，满足强三色不变式。**

Go 1.7 插入屏障的实现代码在 mbarrier.go 中，调用路径为 `writebarrierptr(dst, src)->writebarrierptr_nostore1(dst, src)->gcmarkwb_m(dst, src)`，Go 1.7 在编译的时候，会为全局变量指针/堆指针的写操作插入写屏障相关指令：

以 foo 函数为例：

```go
package main

func main() {
    a := foo();
    println(a)
}

type TT struct{}

type T struct{
    tt *TT
}

func foo() *T {
    a := &T{};
    a.tt = bar() // a 是栈对象，a.tt 是指针，引用了堆对象
    return a
}

func bar() *TT {
    return &TT{}
}
```

`a.tt` 插入写屏障相关指令后的汇编代码如下：

> go build -o aa -gcflags "-N -l" a.go
>
> objdump -d aa \| grep -i '\<main.foo\>' -A 50

![](/assets/img/golang-gc-before-1.8-wb.png)

> 除了在全局变量指针/堆指针的写操作插入屏障相关指令，在 Go Runtime 内部涉及到 memmvoe/memclrHasPointers 的内存操作（map 扩缩容/slice 扩容/chan 发送和接收等）、atomic 包中的交换指针操作、创建新协程的 go 语句（go1.7~1.16 需要对函数参数加入屏障逻辑)）会在启用屏障的时候调用屏障处理函数

需要重新扫描协程栈的 Go 1.7 GC 并发三色垃圾回收算法执行主要流程：

1、GC Start

假设程序执行到下图的状态，开始启动 GC。GC 开始阶段会进行 STW，需要扫描的 GC Root Object 主要在协程栈、Span 上的 finalizer/profile、bss/data 上的全局变量。
该阶段只计算 GC Root Object 扫描的工作量，实际的扫描发生在 Concurrent Mark 阶段。

![](/assets/img/golang-gc-before-1.8.png)

2、GC Concurrent Mark

1\. 依次扫描 data+bss、mspan.specials、G stack 上函数栈帧的局部变量和参数

> 扫描协程栈时，会扫描所有的栈对象，同时安装 stackbarrier，用于记录在并发标记阶段 G 的哪些栈帧执行过了，只有执行过的栈帧，在 Mark Termination 阶段才会被重新扫描

2\. 重新调度运行的 G1 会被加入到 rescan list，堆上新分配的 Object10 直接标记为黑色

3\. 新创建的 G3 会被加入到 rescan list

![](/assets/img/golang-gc-before-1.8-1.gif)

4\. 扫描 Object7 变为黑色

5\. Object7 引用了 Object9，写屏障把 Object9 变为灰色

6\. G3 stack 上的 Object11 引用了 Object6，栈指针上没有写屏障，所以 Object6 保持白色

7\. 删除 Object5 对 Object6 的引用

![](/assets/img/golang-gc-before-1.8-2.gif)

8\. 扫描 Object5 变为黑色

9\. 扫描 Object3 变为黑色

10\. 扫描 Object9 变为黑色

![](/assets/img/golang-gc-before-1.8-3.gif)

3、GC Mark Termination

该阶段需要被扫描的 GC Root Object 只有需要重新扫描的协程栈指向的对象。

1\. rescan G list 出队 G1，扫描 G1 stack

2\. rescan G list 出队 G3，扫描 G3 stack，Object11 变为黑色，Object6 变为灰色

3\. 扫描 Object6 变为黑色

![](/assets/img/golang-gc-before-1.8-4.gif)

4、并发清扫阶段

回收器回收白色对象 Object8，Object4 是栈对象，生命周期是跟随其所在的栈帧的。

Go 1.7 在并发标记结束阶段需要对部分协程栈进行重新扫描的原因就是因为这些协程栈可能引用了不被灰色保护的白色对象（如例子中的 Object11 引用 Object6），从 Go 1.8 开始，Dijkstra 插入写屏障被替换为混合写屏障，在 Dijkstra 插入屏障的基础上加入了 Yuasa 删除屏障，从而解决了重新扫描协程栈的问题，相关的提案：[Proposal: Eliminate STW stack re-scanning](https://github.com/golang/proposal/blob/master/design/17503-eliminate-rescan.md)。

Yuasa 删除屏障伪代码：

```text
writePointer(slot, ptr):
    shade(*slot)
    *slot = ptr
```

**被删除引用的对象（\*slot）如果是白色，则标记为灰色。这确保了白色对象的上游始终有灰色对象，满足弱三色不变式。**

Go 1.18.2 混合写屏障的伪代码，实现可以查看 `src/runtime/asm_amd64.s:TEXT runtime·gcWriteBarrier`：

```text
writePointer(slot, ptr):
    shade(ptr)
    shade(*slot)
    *slot = ptr
```

不需要重新扫描协程栈的 Go 1.18.2 GC 并发三色垃圾回收算法执行主要流程：

1、GC Start

假设程序执行到下图的状态，开始启动 GC。GC 开始阶段会进行 STW，需要扫描的 GC Root Object 主要在协程栈、Span 上的 finalizer、bss/data 上的全局变量。
该阶段只计算 GC Root Object 扫描的工作量，实际的扫描发生在 Concurrent Mark 阶段。

![](/assets/img/golang-gc-1.18.png)

2、GC Concurrent Mark

1\. 依次扫描 data+bss、mspan.specials、G stack 上的函数栈帧的局部变量和参数

> 扫描协程栈时，会扫描所有的栈对象

2\. 重新调度运行 G1，堆上新分配的 Object10 直接标记为黑色

3\. 新创建 G3

![](/assets/img/golang-gc-1.18-1.gif)

4\. 扫描 Object7 变为黑色

5\. Object7 引用了 Object9，插入写屏障把 Object9 变为灰色

6\. G3 stack 上的 Object11 引用了 Object6，栈指针上没有写屏障，所以 Object6 保持白色

7\. 删除 Object5 对 Object6 的引用，删除写屏障将 Object6 变为灰色

![](/assets/img/golang-gc-1.18-2.gif)

8\. 扫描 Object5 变为黑色

9\. 扫描 Object3 变为黑色

10\. 扫描 Object9 变为黑色

11\. 扫描 Object6 变为黑色

![](/assets/img/golang-gc-1.18-3.gif)

3、GC Mark Termination

无需再重新扫描 G1、G3 的堆栈了。

4、并发清扫阶段

回收器回收白色对象 Object8，Object4 和 Object11 是栈对象，生命周期是跟随其所在的栈帧的。

对比 Go 1.7 GC，在删除 Object5 到 Object6 的引用时，Go 1.18 通过 Yuasa 删除写屏障把 Object6 变为了灰色，从而避免了对协程栈的重新扫描。

Yuasa 删除写屏障满足弱三色不变式，又解决了对协程栈的重新扫描问题，那是不是只用 Yuasa 删除写屏障就可以了？答案并不是，插入写屏障也是必须的。如下图，在 Concurrent Mark 阶段，被扫描到的 G1 会被 suspend，而 G2 则在执行，赋值器做了两个操作：

1. 黑色 Object5 引用了白色 Object4
2. 白色 Object3 删除了对白色 Object4 的引用

![](/assets/img/why-need-insert-barrier.gif)

如果没有插入写屏障，Object4 会保持为白色，最后被回收器回收掉。
