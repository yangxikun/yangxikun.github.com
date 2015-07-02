---
layout: post
title: "一种基于哨兵的缓存访问策略"
description: ""
category: 缓存
tags: []
---
{% include JB/setup %}

*学习自[一种基于“哨兵”的分布式缓存设计](http://blog.lichengwu.cn/architecture/2015/06/14/distributed-cache/)*

通常的缓存访问如下，箭头表示访问量，且为同一时刻访问。

![cache access](/assets/img/201507020101.png)

<!--more-->

如果Redis缓存命中，那么web就不会访问数据库，否则，客户端有N个并发请求就会有N个对数据库的并发请求，伴随而来的可能会是N个Redis SET操作。

为了消除这种情况下多余的请求，减轻数据库压力，引入一个“哨兵”请求，即当缓存不命中时，只有一个请求能落到数据库上，其余请求等待缓存更新。

![cache access](/assets/img/201507020102.png)

为了在并发请求中选出一个“哨兵”，对于一个缓存需要有一个计数器与其对应。以下是借助Redis实现的算法流程：

![cache access](/assets/img/201507020103.png)

有一个需要注意的问题就是，上图中红框部分执行失败：例如MySQL无法访问了，那么count将会不断递增，即使MySQL恢复正常了也如此，因为没有请求的count会再次为1。解决办法：

* 手动设置count为0；
* 给count设置上限，当达到上限时设置count为0；
* 将“=1”的条件判断改为类似“count%10=1”；
