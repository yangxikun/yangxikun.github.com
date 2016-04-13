---
layout: post
title: "swoole框架中swoole_client的实现"
description: "swoole框架中swoole_client的实现"
category: PHP
tags: [swoole]
---
{% include JB/setup %}
swoole_client的实现源码主要在swoole-src/swoole_client.c中。

swoole_client有同步和异步两种使用方式，如下：

同步方式：

{% highlight php linenos %}
{% raw %}
<?php
$client = new swoole_client(SWOOLE_SOCK_TCP);
if (!$client->connect('127.0.0.1', 9501, 0.5))
{
    die("connect failed.");
}

if (!$client->send("hello world"))
{
    die("send failed.");
}

$data = $client->recv();
if (!$data)
{
    die("recv failed.");
}

$client->close();
{% endraw %}
{% endhighlight %}

<!--more-->
异步方式：

{% highlight php linenos %}
{% raw %}
<?php
$client = new swoole_client(SWOOLE_SOCK_TCP, SWOOLE_SOCK_ASYNC);

$client->on("connect", function($cli) {
    $cli->send("hello world\n");
});
$client->on("receive", function($cli, $data){
    echo "Received: ".$data."\n";
});
$client->on("error", function($cli){
    echo "Connect failed\n";
});
$client->on("close", function($cli){
    echo "Connection close\n";
});

$client->connect('127.0.0.1', 9501, 0.5);
{% endraw %}
{% endhighlight %}

与swoole_client相关的几个实体如下图：
![](/assets/img/201604130101.png)

以本文开头两个示例代码为例子：

#### 同步方式的执行
- - -
1、执行connect方法

![](/assets/img/201604130102.png)

2、执行send方法，调用swClient_tcp_send_sync()->swConnection_send()->send(/\*系统调用\*/)

3、执行recv方法，调用swClient_tcp_recv_no_buffer()->swConnection_recv()->recv(/\*系统调用\*/)

4、执行close方法，调用swClient_close()

#### 异步方式的执行
- - -
1、执行\_\_construct方法

![](/assets/img/201604130103.png)

2、执行on方法，设置回调函数

3、执行connect方法

![](/assets/img/201604130104.png)

4、执行php_swoole_event_wait函数

![](/assets/img/201604130105.png)