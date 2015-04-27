---
layout: post
title: "理解Linux操作系统——性能度量"
description: ""
category: Linux性能分析
tags: []
---
{% include JB/setup %}
*学习自《Linux性能分析》*

#### 处理器度量
- - -
* CPU utilization: 如果该值超过80%持续一段周期时间，有可能是处理器瓶颈。
* User time: 通常都希望该值越高越好，因为这表明系统在做更多的实际工作。
* System time: 包括了内核操作和中断处理时间，该值应该尽量小。
* Waiting: 花在等待I/O操作出现的CPU时间，该值应该尽量小。
* Idle time: CPU空闲时间。
* Nice time: 花在re-nicing改变进程执行顺序和优先级的时间。
* Load average: 与两个数值密切相关，在等待运行队列的进程数和等待uninterruptible task完成的进程数。该数值应该不大于CPU核数。
* Runable processes: 该值不应该超过CPU核数的10倍。
* Blocked: 进程因为需要等待I/O操作的完成而无法执行。Blocked processes值过大，可能存在I/O瓶颈。
* Context switch: 该值应该尽量小。
* Interrupts: 包括硬中断和软中断，该值过高表明可能软件、内核或驱动存在瓶颈。注意CPU clock产生的中断也是一部分。

<!--more-->

#### 内存度量
- - -
![memory](/assets/img/201504270101.jpg)

* Free memory: 包括buffers和cache。
* Swap usage: Swap In/Out 值过高（200-300/per second）意味着存在内存瓶颈。
* Buffer and cache: 用于文件系统和块设备的缓存。
* Slabs: 内核使用的内存。注意内核页是无法被换出到磁盘的！
* Active versus inactive memory: 提供了系统内存活动信息。

#### 网络接口度量
- - -
* Packets received and sent
* Bytes received and sent
* Collisions per second: 该值应该接近0.
* Packets dropped
* Overruns: 表明网络接口buffer不足的次数。该值应该结合Packets dropped以判断可能的瓶颈是network buffers或network queue length。
* Errors: 错误帧的数目。

#### 块设备度量
- - -
* Iowait: CPU等待I/O操作发生花费的时间，该值应该尽量小。
* Average queue length: 未处理的I/O请求。通常，一个磁盘队列中有2或3个I/O请求是最优的。
* Average wait: 单位ms，包括实际的I/O操作和在I/O queue等待的时间。
* Transfers per second: 每秒有多少I/O操作在执行，该值应该结合kBytes per second以计算出average transfer size，该值通常约等于磁盘子系统的带宽。
* Blocks read/write per second: 每秒读写多少blocks（2.6内核开始是1024bytes）。
* Kilobytes per second read/write
