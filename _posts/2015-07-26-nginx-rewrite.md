---
layout: post
title: "nginx rewrite"
description: ""
category: nginx
tags: [nginx配置]
---
{% include JB/setup %}

*参考[ngx_http_rewrite_module](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html)，对各种rewrite情况进行了实验，总结一下。*

rewrite、location、if的context如下：

* rewrite：server、location、if
* location：server、location
* if：server、location

location匹配规则：

* =：完整的字符串匹配
* ~：大小写敏感的正则匹配
* ~*：大小写不敏感的正则匹配
* ^~：如果最长的前缀匹配有这个修饰符，那么将不再检查正则表达式匹配

location匹配顺序：

1. 检查前缀匹配，记录下最长前缀匹配，如果有^~修饰符，则停止搜索；
2. 检查正则匹配，第一个匹配到正则后停止搜索，如果没有任何正则匹配，那么将使用步骤1中记录的最长前缀。

<!--more-->
