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

那么rewrite规则的路径有如下两种（->代表嵌套、*代表0到n）：

1. server->location*->rewrite；
2. server->location*->if->rewrite。

即：

{% highlight php linenos %}
{
#第一种
    server {
        location {
            location {
                ……
                rewrite规则
            }
        }
    }
#第二种
    server {
        location {
            location {
                ……
                if {
                    rewrite规则
                }
            }
        }
    }
}
{% endhighlight %}

URL中用于匹配和替换的部分是红色字部分：http://rewrite.com`/index/c/i`?hello=world。

rewrite会从配置文件中解析出两个规则集（?代表0或1；+代表1到n）：

1. server->if?->rewrite；
2. server->location+->一堆重写规则集。

{% highlight php linenos %}
{
    server {
        rewriteA
        locationA {
            一堆重写规则集
        }
        if {
            rewriteC
        }
        locationB {
            一堆重写规则集
        }
    }
}
{% endhighlight %}
