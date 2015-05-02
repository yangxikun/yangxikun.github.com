---
layout: post
title: "理解Linux操作系统——分析性能瓶颈"
description: ""
category: Linux性能分析
tags: []
---
{% include JB/setup %}
*学习自《Linux性能分析》*

#### 识别瓶颈
- - -
快速调优策略：

1. 认识你的系统。
2. 备份系统。
3. 监控和分析系统性能。
4. 缩小瓶颈范围，找到根源。
5. 通过每次只修改一个地方来解决瓶颈问题。
6. 回到第3步直到对系统的性能满意为止。

应该记录下调优的操作，特别是对性能有影响的操作。

<!--more-->

#### 收集信息
- - -
通常，你能得到的第一手信息就是关于问题的描述。对问题进行探索性地提问和记录是非常重要的。这里有一些问题有助于你对系统有一个更好的了解：

* 服务器系统类型、版本、配置是什么？
* 问题的症状，给出了哪些错误信息，日志记录？
* 谁遇到了这个问题？
* 问题是否能被重现？重现的步骤、定期出现还是不定期
* 之前对系统做了哪些更改可能导致问题的发生？
* 问题的紧急程度如何，需要何时解决？

#### 分析系统性能
- - -
如果是应用程序本身的问题：

* 自己开发的？debug吧
* 开源软件？看是否配置不正确、需要升级，或提issue
* 商业软件？看是否配置不正确、需要升级，或找售后服务

如果是系统本身或硬件问题：

**CPU瓶颈**

查找瓶颈

* uptime 查看负载。
* top 查看CPU使用率，以及哪个进程占用高。
* sar 需要安装sysstat，可查看系统一段时间内的运行情况。

通常以上命令的输出信息可以从运维平台获取到。

调优

* 确保没有运行不需要的程序。`ps -ef`，停止或将其改为定时任务，在非峰值时间运行。
* 通过renice调整进程priority。
* 绑定进程到CPU上。
* 考虑纵向扩展CPU（适合单线程程序）还是横向扩展（适合多线程程序）。

**内存瓶颈**

Memory available

> 还有多少内存可以使用，执行`free -l -t -o`查看。

Page faults

> 两种类型：soft page faults（在内存中找到page）和hard page faults（内存中找不到page，必须从磁盘中获取）。
> 执行`sar -B`可查看页面交换情况。

File system cache

> 用于文件系统缓存的内存，执行`free -l -t -o`查看。

Private memory for process

> 进程占用的内存，执行`pmap`查看。

调优

* 调优swap space。
* 增加或减少页面大小。
* 改善对active和inactive内存的处理。
* 调整page-out rate。
* 停止不需要的程序。
* 增加内存。

**磁盘瓶颈**

查找瓶颈

如果服务器表现出如下症状，很可能是遇到磁盘瓶颈：

* 低速磁盘可能会导致：

> Memory buffers充满了write data（或waiting for read data），这将会使write request（或the response is waiting for read data in the disk queue）。
> 在内存不足的情况下，在没有足够的内存提供给network requests的情况下，会导致同步的磁盘I/O。

* Disk utilization，controller utilization之一或都会非常高.
* 在磁盘I/O完成之后，许多LAN传输才会发生，导致非常长的响应时间和较低的network utilization。
* 磁盘I/O会花费许多时间，并且磁盘队列也会变得饱满，所以CPU将会空闲，或者较低的使用率。

磁盘访问是随机还是顺序？是large I/O还是small I/O？这些问题的答案有助于磁盘子系统的调优。

![disk I/O](/assets/img/201504290101.png)

随机读写通常需要多块磁盘更好，例如大型数据库。

顺序读写通常需要关注磁盘子系统的bus bandwidth。

vmstat和iostat可用于查看磁盘I/O情况。

调优

* 如果是随机读写，考虑加磁盘；如果是顺序读写，考虑增加更快的磁盘控制器。
* 合理的使用RAID，以及选择文件系统。
* 增加RAM。

**网络瓶颈**

* Packets received、Packets sent：网络接口上的收发包
* Collision packets：当一个域里有太多系统时会发生冲突。Hub的使用可能导致过多的冲突。
* Dropped packets：被丢弃的包。
* Errors：通信线路质量不好。
* Faulty adapters：网络硬件出问题了。

调优

* 确保网卡配置跟路由器、交换机配置吻合（例如frame size）。
* 调整网络拓扑。
* 使用更快的网卡。
* 调整IPV4 TCP内核参数。
