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

以上配置构成了如下两个规则集：

规则集1：

* server->rewriteA
* server->if->rewriteC

规则集2：

* server->locationA->一堆重写规则集
* server->locationB->一堆重写规则集

rewrite重写步骤：

1、根据规则集1中的规则先后顺序重写，如果遇到（break/last）则进入步骤2

2、根据规则集2中的location先后顺序匹配URL：

> 如果无任何匹配，则进入3，如果匹配到某个location，则根据location中的location或规则，进行匹配或重写URL

>> location中发生重写时，如果遇到last或者执行到所在location的结尾（发生过重写），则进入步骤2，如果遇到break或者没有发生重写，则进入步骤3

3、结束rewrite。
