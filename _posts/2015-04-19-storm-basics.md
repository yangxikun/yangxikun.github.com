---
layout: post
title: "Storm 基础知识"
description: ""
category: Storm
tags: []
---
{% include JB/setup %}

*官方手册：[Storm Manual](https://storm.apache.org/documentation/Documentation.html)*

#### Storm Cluster 集群
- - -
节点：快速失败、无状态

* master node: Nimbus，负责分发代码到集群中，为机器分配工作，监控异常。
* worker node: Supervisor，监听Nimbus为其分配工作，运行工作进程。
* Nimbus和Supervisor之间的交互通过Zookeeper集群协调，它们的状态都保存在Zookeeper或者本地磁盘上。

![storm cluster](/assets/img/201504190101.png)

<!--more-->

#### Topologies
- - -
A graph of spouts and bolts that are connected with stream groupings.

Streams：An unbounded sequence of tuples that is processed and created in parallel in a distributed fashion.

![storm topologies](/assets/img/201504190102.png)

#### 消息处理可靠性保证
- - -
Storm提供数据至少被处理一次的可靠性保证，由编程人员决定是否使用这一特性。

Spout所发射出的Tuple1是会衍生出更多的Tuple*，如果在Bolot emit时进行anchor，那么将会构成一棵Tuple树，只有当整棵Tuple树被正确处理了，才算Tuple1被完整地正确处理了。

如果要确保数据至少被处理一次：

* 在Spout emit时添加一个MsgID，那么ack和fail方法将会被调用当Tuple被正确地处理了或发生了错误。`_collector.emit(new Values("field1", "field2", 3) , msgId);`
* 在Bolt emit时进行anchor。` _collector.emit(tuple, new Values(word));`

如果对数据可靠性要求不高，那么可以通过如下方法去除可靠性：

* 把Config.TOPOLOGY_ACKERS设置成0。在这种情况下，Storm会在Spout发射一个Tuple之后马上调用Spout的ack方法。也就是说这个Tuple树不会被跟踪。
* 在Tuple层面去掉可靠性。 你可以在发射Tuple的时候不指定MessageID来达到不不跟踪这个Tuple的目的。
* 如果你对于一个Tuple树里面的某一部分到底成不成功不是很关心，那么可以在发射这些Tuple的时候unanchor它们。这样这些Tuple就不在Tuple树里面，也就不会被跟踪了。

两个相关的类：BaseBasicBolt类封装了自动anchor和发送ack的逻辑，而BaseRichBolt没有。

BaseRichBolt：

> You must – and are able to – manually ack() an incoming tuple.
> Can be used to delay acking a tuple, 
> e.g. for algorithms that need to work across multiple incoming tuples.

BaseBasicBolt：

> Auto-acks the incoming tuple at the end of its execute() method.
> These bolts are typically simple functions or filters.

#### Storm并发度
- - -
Running Topology 组成：
1. worker process
1. executor
1. Task

一个Topology可以包含一个或多个worker，一个worker对应于一个特定的Topology。

![topology concurrent](/assets/img/201504190103.png)

parallelism hint： the initial number of executor (threads) of a component.
If you do not explicitly configure the number of tasks, Storm will run by default one task per executor.

![topology concurrent](/assets/img/201504190104.png)
