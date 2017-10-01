---
layout: post
title: "ES简单的通用日志方案"
description: ""
category: 
tags: []
---
{% include JB/setup %}
本文介绍笔者所使用的ES简单通用的应用日dudp志方案。

首先说明下需求：业务日志希望能够有一个地方能够统一存储，能够检索和展示报表，并且简便地接入新的业务日志。

于是设计了如下的架构：

![](/assets/img/201710010101.png)

1. 业务通过udp协议的方式将日志以json编码的格式发送给udp2kafka服务，并且日志中包含需要写入的index
1. udp2kafka将接收到的日志写入kafka
1. kafka2es进行消费，根据日志中的index，写入到es相应的index中

<!--more-->

##### 为何采用udp协议？

udp协议是无连接的，通过udp发送日志对应用性能影响较小。虽然udp可能会丢包，但对于日志来说是可接受少量丢包的，并且如果udp2kafka与业务部署在同一机房，基本是不会丢包的，目前从我的使用情况看没发现有问题。

##### 为何采用json编码的日志？

主要是为了能够使udp2kafka、kafka2es足够简单通用，通用的东西都会有一定的规范，业务必须在每条日志中包含`index`字段，表明要写入到es中的哪个index，其他字段业务根据需要自定义，另外json格式的日志能够方便kafka2es的解析。

##### udp协议传输json编码的日志问题？

目前我的实现是业务需要把一个完整的json编码的日志作为单独的一个udp包发送，不能拆分为多个udp包，也不能在一个udp包中包含多个日志。当然，这取决于udp2kafka的实现。所以这样就会限制了单条业务日志的大小，最大为（65535-20-8）字节。

一开始udp2kafka是用logstash5.4.1+logstash-output-kafka-4.0.4（为了支持0.9的kafka集群）做的，跑在K8S上，但在线上运行一段时间之后，就会有几个实例出问题，并且导致日志丢失，logstash也没有错误的日志输出，strace看到像是jvm线程死锁了。一开始采用重启实例的方法解决，但问题一直存在，于是就自己用golang重写了udp2kafka，目前来看运行得很稳定，而且内存占用只是logstash的1/10（对logstash的内存使用表示惊讶= =）。

##### 为何多加一层kafka？

因为业务日志并发产生过多，如果由udp2kafka直接写入ES的话，ES的写入将成为瓶颈，可能会导致日志丢失。所以加多一层kafka作为缓冲，通过调整kafka2es的实例数可以稳定控制写入ES的流量。

##### 动态映射（Dynamic Mapping）和索引模板（Index Templates）

通过[动态映射](https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic-mapping.html)和[索引模板](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-templates.html#indices-templates)，我为业务日志定义了一套简单的映射规则，如下所示`GET _template/qssweb-business-log`：

```json
{
  "qssweb-business-log": {
    "order": 0,
    "template": "qssweb-business-log-*",
    "settings": {},
    "mappings": {
      "_default_": {
        "dynamic_templates": [
          {
            "analyzed_string": {
              "mapping": {
                "search_analyzer": "whitespace",
                "analyzer": "ik_max_word",
                "type": "text",
                "fields": {}
              },
              "match_mapping_type": "*",
              "match": "*_analyzed"
            }
          },
          {
            "not_analyzed_string": {
              "mapping": {
                "type": "keyword",
                "fields": {}
              },
              "match_mapping_type": "string",
              "match": "*"
            }
          }
        ],
        "_all": {
          "enabled": false
        }
      }
    },
    "aliases": {}
  }
}
```
如果日志字段名称以`_analyzed`结尾，那么将会被解析为text类型以支持全文检索，analyzer用于设置字段在建立全文索引时的分词方式（[ik](https://github.com/medcl/elasticsearch-analysis-ik)能够对中文进行分词），search_analyzer则用于设置对该字段进行检索时，查询词的分词方式。其他的string类型的日志字段会被解析为keyword类型，只支持精确匹配的检索。

精确值与全文的区别：[https://github.com/elasticsearch-cn/elasticsearch-definitive-guide/blob/cn/052_Mapping_Analysis/30_Exact_vs_full_text.asciidoc](https://github.com/elasticsearch-cn/elasticsearch-definitive-guide/blob/cn/052_Mapping_Analysis/30_Exact_vs_full_text.asciidoc)

##### kafka2es

kafka2es使用的logstash5.4.1+logstash-input-kafka-4.2.0，目前来看运行稳定，不过还是占的内存较多，其配置如下：

```plain
input {
        kafka {
                bootstrap_servers => "10.213.166.137:9092"
                auto_offset_reset => "earliest"
                topics => ["qlog_qssweb_general_udp_log"]
                codec => "json"
        }
}
filter {
        # 解析日志时间
        date {
                match => ["time", "UNIX_MS"]
        }
        # 将index添加到@metadata中，这样不会被写入到es
        # 同时删除无用字段
        mutate {
                add_field => { "[@metadata][index]" => "%{index}" }
                remove_field => [ "@version", "host", "index", "time"]
        }
}
output {
        # 按日期分隔日志
        elasticsearch {
                index => "%{[@metadata][index]}-%{+YYYY.MM.dd}"
                hosts => ["10.226.176.242:9016"]
                user => "user"
                password => "pass"
        }
}
```