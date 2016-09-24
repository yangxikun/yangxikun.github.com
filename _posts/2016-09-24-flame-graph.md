---
layout: post
title: "初识火焰图"
description: "火焰图"
category: Linux性能分析
tags: []
---
{% include JB/setup %}

本文学习自：[白话火焰图](http://huoding.com/2016/08/18/531)

### 火焰图的分类
- - -

压测程序时，火焰图能够帮助我们直观地了解到程序的性能瓶颈。本文主要是记录下在生成火焰图的时候的一些注意事项。以下链接是对每个类别的火焰图的详细说明。

1. On-CPU：[CPU Flame Graphs](http://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html)
1. Off-CPU：[Off-CPU Flame Graphs](http://www.brendangregg.com/FlameGraphs/offcpuflamegraphs.html)
1. Memory：[Memory Leak (and Growth) Flame Graphs](http://www.brendangregg.com/FlameGraphs/memoryflamegraphs.html)
1. Hot/Cold：[Hot/Cold Flame Graphs](http://www.brendangregg.com/FlameGraphs/hotcoldflamegraphs.html)
1. Differential：[Differential Flame Graphs](http://www.brendangregg.com/blog/2014-11-09/differential-flame-graphs.html)

关于火焰图的PPT（讲解得非常详细）：[Blazing Performance with Flame Graphs](http://www.slideshare.net/brendangregg/blazing-performance-with-flame-graphs)

<!--more-->

### 如何生成火焰图
- - -

首先是需要在程序运行时，对其进行采样。linux下常用的有perf和systemtap，我这里用了systemtap来做实验，因为已经牛人写好了一些systemtap相应的脚本可以直接拿来用了：[nginx-systemtap-toolkit](https://github.com/openresty/nginx-systemtap-toolkit)。

systemtap依赖的安装：

1、CentOS

```c
shell> yum install yum-utils
shell> yum install kernel-devel
shell> debuginfo-install kernel
shell> yum install systemtap
```
在执行`debuginfo-install kernel`的时候，没能安装成功，于是就直接到[http://debuginfo.centos.org/7](yum install systemtap)下载相应的包手动安装了。

2、Debian

```c
shell> apt-get install systemtap linux-image-`uname -r`-dbg linux-headers-`uname -r`
```

systemtap的工作原理：

```c
systemtap脚本——>翻译为C程序——>编译为内核模块——>安装到内核上运行，开始采样——>结束采样，从内核卸载
```

我用[nginx-systemtap-toolkit](https://github.com/openresty/nginx-systemtap-toolkit)提供的两个perl脚本（它们会根据参数生成相应的systemtap脚本并执行）做了实验：

* sample-bt：用来生成 On-CPU 火焰图的采样数据
* sample-bt-off-cpu：用来生成 Off-CPU 火焰图的采样数据

注意事项：

1、普通用户身份无法执行systemtap。

2、什么时候开始采样？

当执行`sample-bt -p 50417 -t 5 -u  > sample.bt`的时候，需要等到输出类似`Tracing pid (exec_path) in kernel|user -space...`的时候才真正开始采样。可以为sample-bt加多一个参数`-a '-vv'`就可以看到一些debug输出。

3、理解On-CPU采样的工作原理

原理是定时地判断当前cpu上运行的进程是否为要采样的进程，如果是的话就根据设置的采样点进行采样。

如果被采样的进程一直处于就绪状态，没有被调度执行，那么你将会看到sample-bt的输出如下面这样：

![](http://km.oa.com/files/photos/pictures/201609/1474705802_98_w522_h150.png)

为何没有停止呢？这就需要从sample-bt生成的systemtap脚本看看有没什么线索了：

执行`sample-bt -p 114356 -t 5 -u -a '-vv' -d `，这里多加了一个`-d`参数，让sample-bt输出生成的systemtap脚本：

```c
/*
target()是systemtap的函数库Tapset提供的，用于获取当前采样的进程pid
*/
probe begin {
    warn(sprintf("Tracing %d (/usr/sbin/mysqld) in user-space only...\n", target()))
}


global bts;
global quit = 0;

probe timer.profile {//每个一段时间被调用一次
    if (pid() == target()) {//当前运行的进程与被采样的进程是否为同一个
        if (!quit) {
            bts[ubacktrace()] <<< 1;

        } else {
            //采样停止
            foreach (bt in bts- limit 1024) {
                print_ustack(bt);
                printf("\t%d\n", @count(bts[bt]));
            }

            exit()
        }
    }
}

probe timer.s(5) {//5秒后执行
    nstacks = 0
    foreach (bt in bts limit 1) {
        nstacks++
    }

    if (nstacks == 0) {//没采样到数据直接退出
        warn("No backtraces found. Quitting now...\n")
        exit()

    } else {//采样到数据了，设置停止标志
        warn("Time's up. Quitting now...(it may take a while)\n")
        quit = 1
    }
}
```

* [SystemTap Tapset Reference Manual](https://sourceware.org/systemtap/tapsets/)
* [SystemTap Language Reference](https://sourceware.org/systemtap/langref/langref.html)

所以上图中重复的输出了`Time's up. Quitting now...(it may take a while)`是因为开始采样后，timer.profile采样到数据，timer.s设置了停止标志之后，在timer.profile执行时一直没遇到被采样的进程被调度执行。此时只需要让这个进程被调度执行就行了，比如：如果该进程是一个服务，可以发送个请求给它。

对于Off-CPU采样的工作原理，同样可以使用`sample-bt-off-cpu -a '-vv' -u -p 114356 -t 5 -d`查看生成的systemtap脚本。


生成火焰图的步骤：

1. 采样：`sample-bt -p 92869 -t 10 -u -a '-vv' > a.bt`
1. 转化为火焰图：`stackcollapse-stap.pl a.bt > a.cbt;flamegraph.pl a.cbt > a.svg`

测试时生成的一个On-CPU的图片，从图中可看出频繁的tcp connect和close是目前的瓶颈之一：
![](/assets/img/201609240101.png)