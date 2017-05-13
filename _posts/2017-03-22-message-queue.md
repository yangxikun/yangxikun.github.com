---
layout: post
title: "关于消息队列的思考"
description: "关于消息队列的思考"
category: 
tags: [消息队列]
---
{% include JB/setup %}
消息队列是服务架构中常见的组件，可用于服务间解耦、事件广播、任务异步/延迟处理等，本文对于消息队列的实现如何满足几种消费语义进行了阐述。

### 消息队列组成
* 生产者（Producer）：负责产生消息
* 消息代理（Message Broker）：负责存储/转发消息（转发分为推和拉两种，拉是指Consumer主动从Message Broker获取消息，推是指Message Broker主动将Consumer感兴趣的消息推送给Consumer）
* 消费者（Consumer）：负责消费消息

![](/assets/img/201703220101.png)

<!--more-->

### 消息队列的消费语义
1. 消息至多被消费一次
1. 消息至少被消费一次
1. 消息仅被消费一次

为了支持上面3种消费语义，可以分3个阶段考虑消息队列系统中Producer、Message Broker、Consumer需要满足的条件：

#### 1、消息至多被消费一次

该语义是最容易满足的，特点是整个消息队列吞吐量大，实现简单。适合能容忍丢消息，消息重复消费的任务。

* **Producer发送消息到Message Broker阶段**：Producer发消息给Message Broker，不要求Message Broker对接收到的消息响应确认，Producer也不用关心Message Broker是否收到消息了。
* **Message Broker存储/转发阶段**：对Message Broker的存储不要求持久性，转发消息时也不用关心Consumer是否真的收到了。
* **Consumer消费阶段**：Consumer从Message Broker中获取到消息后，可以从Message Broker删除消息，或Message Broker在消息被Consumer拿去消费时删除消息，不用关心Consumer最后对消息的消费情况如何。

#### 2、消息至少被消费一次

适合不能容忍丢消息，允许重复消费的任务。

* **Producer发送消息到Message Broker阶段**：Producer发消息给Message Broker，Message Broker必须响应对消息的确认。
* **Message Broker存储/转发阶段**：Message Broker必须提供持久性保障，转发消息时，Message Broker需要Consumer通知删除消息，才能将消息删除。
* **Consumer消费阶段**：Consumer从Message Broker中获取到消息，必须在消费完成后，Message Broker上的消息才能被删除。

#### 3、消息仅被消费一次

适合对消息消费情况要求非常高的任务，实现较为复杂。

在这里需要考虑一个问题，就是这里的“仅被消费一次”指的是如下哪种场景：

* Message Broker上存储的消息被Consumer仅消费一次
* Producer上产生的消息被Consumer仅消费一次

**Message Broker上存储的消息被Consumer仅消费一次** 场景要求：

* **Producer发送消息到Message Broker阶段**：Producer发消息给Message Broker，不要求Message Broker对接收到的消息响应确认，Producer也不用关心Message Broker是否收到消息了。
* **Message Broker存储/转发阶段**：Message Broker必须提供持久性保障，并且每条消息在其消费队列里有唯一标识（这个唯一标识可以由Producer产生，也可以由Message Broker产生）。
* **Consumer消费阶段**：Consumer从Message Broker中获取到消息后，需要记录下消费的消息标识，以便在后续消费中防止对某个消息重复消费（比如Consumer获取到消息，消费完后，还没来得及从Message Broker删除消息，就挂了，这样Message Broker如果把消息重新加入待消费队列的话，那么这条消息就会被重复消费了）。

**Producer上产生的消息被Consumer仅消费一次** 场景要求：

* **Producer发送消息到Message Broker阶段**：Producer发消息给Message Broker，Message Broker必须响应对消息的确认，并且Producer负责为该消息产生唯一标识，以防止Consumer重复消费（因为Producer发消息给Message Broker后，由于网络问题没收到Message Broker的响应，可能会重发消息给到Message Broker）。
* **Message Broker存储/转发阶段**：Message Broker必须提供持久性保障，并且每条消息在其消费队列里有唯一标识（这个唯一标识需要由Producer产生）。
* **Consumer消费阶段**：Consumer从Message Broker中获取到消息后，需要记录下消费的消息标识，以便在后续消费中防止对某个消息重复消费（比如Consumer获取到消息，消费完后，还没来得及从Message Broker删除消息，就挂了，这样Message Broker如果把消息重新加入待消费队列的话，那么这条消息就会被重复消费了）。

### 结语

现在业内已经有许多成熟的消息队列的实现了，对于选择用哪一个实现，可以先根据业务需要支持的消费语义进行初步筛选，之后再根据运维难度、社区活跃度、性能、可用性等综合考虑选择合适的消息队列系统，如何判断一个消息队列实现是否支持某个消费语义，根据本文中阐述的3个阶段去判断即可。
