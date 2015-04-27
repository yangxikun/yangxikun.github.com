---
layout: post
title: "理解Linux操作系统——Process和Memory"
description: ""
category: Linux性能分析
tags: []
---
{% include JB/setup %}
*学习自《Linux性能分析》*

#### Linux 进程管理
- - -
Process：an instance of execution that runs on a processor.

Linux操作系统通过进程描述符task_struct结构体管理进程。

![process task_struct](/assets/img/201504200101.png)

<!--more-->

Thread： an execution unit generated in a single process, also called Light Weight Process (LWP).

![Thread](/assets/img/201504200102.png)

#### 进程priority和nice级别
- - -
priority: 包括dynamic priority和static priority，值越大，获得CPU的可能性也越大。

nice: 用户可以改变进程的nice级别，来间接地改变进程的static priority。取值范围19（lowest priority）到-20（highest priority），默认值0。

#### 上下文切换
- - -
在进程执行时，运行进程的信息存储在处理器的寄存器和cache中。上下文切换只发生在内核态。

处理器的3种状态：

* 内核态：运行于Process Context，内核代表进程运行于内核空间
* 内核态：运行于Interrupt Context，内核代表硬件运行于内核空间
* 用户态：运行于用户空间

![context switch](/assets/img/201504200103.png)

#### 中断处理
- - -
hard interrupt: 由硬件产生。

soft interrupt: 由程序产生，即信号。

在多处理器环境下，通过绑定中断到一个特定的处理器上能够提高系统的性能。

#### 进程状态
- - -
* TASK_RUNNING: 正在运行或者在运行等待队列中。
* TASK_STOPPED: 接收到了信号（例如SIGINT、SIGSTOP）使进程挂起，等待SIGCONT信号恢复运行。
* TASK_INTERRUPTIBLE: 进程被挂起，并等待特定的条件发生。
* TASK_UNINTERRUPTIBLE: 信号对于处于该状态的进程无作用，例如一个在等待磁盘I/O操作de进程。
* TASK_ZOMIBE: 进程调用exit()后，等待父进程获取其退出状态信息。

![process state](/assets/img/201504200104.png)

#### 进程内存段
- - -
进程地址空间：

![process address space](/assets/img/201504200105.png)

* Text segment: 存储可执行代码。
* Data segment: 
> Data: 初始化数据，例如静态变量
> BSS: zero-initialized data
> Heap: malloc()分配的内存，往高地址空间增长
* Stack segment: local variables、函数参数、函数的返回地址，往低地址空间增长。

可通过pmap命令查看用户进程地址空间情况。

#### Linux CPU 调度
- - -
Linux内核使用O(1)算法来选择一个进程运行。

算法使用了2个进程优先级数组：active和expired。

当一个进程次分配到时间片时，它会被放置在active数组。当进程时间片用完时，会分配新的时间片，然后放置在expired数组。当active数组为空了，两个数组就进行转换。

![cpu](/assets/img/201504200106.png)

#### 内存架构
- - -
*参考博文：[linux内核内存管理](http://wushank.blog.51cto.com/3489095/1406480)和[linux内存管理--用户空间和内核空间](http://blog.csdn.net/yusiguyuan/article/details/12045255)*

通常在32bit机器上，Linux会把物理内存划分为几个zone：ZONE_DMA（I/O设备专用内存）、ZONE_NORMAL（内核能直接使用）、ZONE_HIGHMEM（内核无法直接使用，需要将这部分空间做一个映射，映射关系存储在ZONE_NORMAL中）。

![memory zone](/assets/img/201504210101.png)

![memory zone](/assets/img/201504210104.png)

虚拟地址被划分为内核空间和用户空间，32bit机器上虚拟地址与物理地址映射关系：

![virtual to physical](/assets/img/201504210102.jpg)

32bit机器4G虚拟地址空间解析图：

![virtual 32bit](/assets/img/201504210103.png)

对于64bit机器，通常没有ZONE_HIGHMEM，因为64bit的虚拟地址中的内核空间足以映射非常大的物理内存。

![virtual 64bit](/assets/img/201504210106.png)

#### 虚拟内存管理器
- - -
![virtual to physical](/assets/img/201504210105.png)

默认的虚拟内存管理器配置会把所有空闲的内存分配为disk cache使用。

#### 页面分配
- - -
一页通常为4K bytes。当一个进程请求一定数量的页面时，如果有足够的页面，Linux内核会立即分配给它。否则，需要从其他进程或页缓存获取页面。

#### 伙伴系统
- - -
Linux内核通过buddy system的机制维护空闲页面。

#### 页面再生
- - -
如果无法为进程分配到需要的页面，Linux内核会试图通过释放某些页面来为进程分配所需的页面。这个过程称为page reclaiming，kswapd内核线程和try_to_free_page()内核函数负责这一过程。

kswapd通常是一个处于interruptible state的线程，被buddy system调用，基于LRU算法查找候选的页面。它会扫描active list查找出最近最少使用的页面，然后将其放到inactive list中。

可通过`vmstat -a`命令查看有多少内存是active和inactive。

kswapd也遵循另外一个原则。页面通常用于两种途径：page cache和process address space。当kswapd进行page reclaiming时，会选择page cache进行释放，而不是page out或者swap out进程的页面。

* page out: take some pages (a part of entire address space) into swap space.
* swap out: taking entire address space into swap space.

#### swap
- - -
Linux会高效地利用swap空间，当发现swap空间使用了许多时，并不一定是内存瓶颈问题。
