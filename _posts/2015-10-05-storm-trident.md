---
layout: post
title: "Storm Trident 学习"
description: ""
category: Storm
tags: []
---
{% include JB/setup %}
Storm支持的三种语义：

1. 至少一次
1. 至多一次
1. 有且仅有一次

#### 至少一次语义的Topology写法
- - -
参考资料：[Storm消息的可靠性保障](http://ifeve.com/storm-guaranteeing-message-processing/#header)
Storm提供了Acker的机制来保证数据至少被处理一次，是由编程人员决定是否使用这一特性，要使用这一特性需要：

* 在Spout emit时添加一个MsgID，那么ack和fail方法将会被调用当Tuple被正确地处理了或发生了错误。`_collector.emit(new Values("field1", "field2", 3) , msgId);`
* 在Bolt emit时进行锚定。`_collector.emit(tuple, new Values(word));`

<!--more-->

Spout例子：
{% highlight java linenos %}
public class RandomSentenceSpout extends BaseRichSpout {
    private SpoutOutputCollector _collector;
    private Random _rand;
    private ConcurrentHashMap<UUID, Values> _pending;

    public void open(Map map, TopologyContext topologyContext, SpoutOutputCollector spoutOutputCollector) {
        _collector = spoutOutputCollector;
        _rand = new Random();
        _pending = new ConcurrentHashMap<UUID, Values>();
    }

    public void nextTuple() {
        Utils.sleep(1000);
        String[] sentences = new String[] {
                "I write php",
                "I learning java",
                "I want to learn swool and tfs"
        };
        String sentence = sentences[_rand.nextInt(sentences.length)];
        Values v = new Values(sentence);
        UUID msgId = UUID.randomUUID();
        this._pending.put(msgId, v);
        _collector.emit(v, msgId);//发射tuple时，添加msgId
    }

    public void declareOutputFields(OutputFieldsDeclarer outputFieldsDeclarer) {
        outputFieldsDeclarer.declare(new Fields("world"));
    }

    public void ack(Object msgId) {
        this._pending.remove(msgId);//当消息被完整性地处理了
    }

    public void fail(Object msgId) {
        this._collector.emit(this._pending.get(msgId), msgId);//当消息处理失败了，重新发射，当然也可以做其他的逻辑处理
    }
}
{% endhighlight %}

Bolt例子：
{% highlight java linenos %}
public class SplitSentence extends BaseRichBolt {
    OutputCollector _collector;

    public void prepare(Map map, TopologyContext topologyContext, OutputCollector outputCollector) {
        _collector = outputCollector;
    }

    public void execute(Tuple tuple) {
        String sentence = tuple.getString(0);
        for (String word : sentence.split(" "))
            _collector.emit(tuple, new Values(word));//发射tuple时进行锚定
        _collector.ack(tuple);//对处理完的tuple进行确认
    }

    public void declareOutputFields(OutputFieldsDeclarer outputFieldsDeclarer) {
        outputFieldsDeclarer.declare(new Fields("word"));
    }
}
{% endhighlight %}

#### 至多一次的语义
- - -
如果只需要至多一次的语义，那么只需要在编程的时候不用Storm的Acker机制，可以通过如下方法达到：

* 把Config.TOPOLOGY_ACKERS设置成0。在这种情况下，Storm会在Spout发射一个Tuple之后马上调用Spout的ack方法。也就是说这个Tuple树不会被跟踪。
* 在Tuple层面去掉可靠性。 你可以在发射Tuple的时候不指定MessageID来达到不不跟踪这个Tuple的目的。
* 如果你对于一个Tuple树里面的某一部分到底成不成功不是很关心，那么可以在Bolt发射这些Tuple的时候不锚定它们。这样这些Tuple就不在Tuple树里面，也就不会被跟踪了。

需要注意的点：每个待处理的 tuple 都必须显式地应答（ack）或者失效（fail）。因为 Storm 是使用内存来跟踪每个 tuple 的，所以，如果你不对每个 tuple 进行应答或者失效，那么负责跟踪的任务很快就会发生内存溢出。

#### 有且仅有一次的语义
- - -
参考资料：[Trident State](http://ifeve.com/storm-trident-state/#header)

Trident中的三种Spout，根据Spout代码情况可判断属于哪个：

事务型Spout的特点：

* 每个 batch 的 txid 永远不会改变。对于某个特定的 txid，batch 在执行重新处理操作时所处理的 tuple 集和它的第一次处理操作完全相同。
* 不同 batch 中的 tuple 不会出现重复的情况（某个 tuple 只会出现在一个 batch 中，而不会同时出现在多个 batch 中）。
* 每个 tuple 都会放入一个 batch 中（处理操作不会遗漏任何的 tuple）。

模糊型Spout的特点：

* 不同 batch 中的 tuple 不会出现重复的情况（某个 tuple 只会出现在一个 batch 中，而不会同时出现在多个 batch 中）。
* 每个 tuple 都会放入一个 batch 中（处理操作不会遗漏任何的 tuple）。
* 每个 tuple 都会通过某个 batch 处理完成。不过，在 tuple 处理失败的时候，tuple 有可能继续在另一个 batch 中完成处理，而不一定是在原先的 batch 中完成处理。

非事务型Spout的特点：

* 非事务型 spout 有可能提供一种“至多一次”的处理模型，在这种情况下 batch 处理失败后 tuple 并不会重新处理。
* 也有可能提供一种“至少一次”的处理模型，在这种情况下可能会有多个 batch 分别处理某个 tuple。

假如你的拓扑需要计算单词数，而且你准备将计数结果存入一个 K-V 型存储系统中。这里的 key 就是单词，value 对应于单词数。如果仅仅存储计数结果是无法确定某个batch中的tuple是否已经被处理过的，例如一个batch已经处理到最后一步并成功写入到了存储系统中，正准备Ack时挂了，那么重新发射这个batch后，被完整地处理了，batch中的tuple就被处理了多次，为了解决这个问题需要State的帮助，即存储的时候做一些处理。

Trident中的三种State，Storm已经提供了多中存储系统State的实现，如MemoryMapState、MemcachedState：

事务型State

* 特点：对于特定的 txid 永远只与同一组 tuple 相关联。
* 实现方式：将单词、txid与计数值一起存入数据库，每次更新时，比较txid是否相同，如果不相同则更新，否则不更新。

模糊型State

* 特点：为了配合模糊型Spout达到有且仅有一次的语义而设计。
* 实现方式：将单词、前一个结果值（preValue）、计数值和txid一起存入数据库，每次更新时，比较txid，若不同，则记录更新为`(单词, 存储的计数值作为preValue, 存储的计数值+batch计算出来的值作为计数值, batch的txid)`；若相同，则记录更新为`(单词, preValue, preValue加上batch计算出来的值作为计数值, txid)`。

非事务型State

* 特点：不能保证有且仅有一次的语义
* 实现方式：数据库只记录了单词，计数值

不同的Spout类型与不同的State类型的组合是否支持有且仅有一次的消息处理语义：
![](/assets/img/201510050101.png)

上图中的Yes表示组合支持有且仅有一次的语义。
