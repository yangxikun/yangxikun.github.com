---
layout: post
title: "swoole_server SWOOLE_PROCESS模式执行探讨"
description: "swoole_server SWOOLE_PROCESS模式执行探讨"
category: PHP
tags: [swoole]
---
{% include JB/setup %}
swoole_server的SWOOLE_PROCESS模式是一个比较复杂的运行模式，因为存在了大量的进程间通信，复杂的连接管理等。

其工作模式大致如下图，这里省略掉Manager、Task进程：
![](/assets/img/201604160101.png)

<!--more-->
master进程负责监听服务端口，reactor thread作为客户端与worker进程间数据交互的代理。（nginx的话是worker进程去抢占accept客户端连接，后来支持reuse_port，linux内核直接分配新的连接到worker，worker不需要再抢占了，对性能有一定的提升，swoole_server的SWOOLE_BASE模式也支持reuse_port）

swoole_server运行时与其相关的数据结构太多了= =，这里就只简单讲讲几个关键的：

1. swServer：服务器相关配置（reactor_num, worker_num, dispatch_mode）、监听端口列表、连接列表、各类回调函数（onStart, onShutdown, onWorkerStart）；
1. Factory：服务器的运行模式，实际上只有SWOOLE_BASE（ReactorProcess）和SWOOLE_PROCESS（ReactorThread）两种，根据不同模式，Factory分别会是swFactory或swFactoryProcess；
1. swReactorThread：SWOOLE_PROCESS模式下，在master进程中由线程负责处理客户端连接；
1. swWorker：worker进程的信息；
1. swServerG：存储当前进程的信息；
1. swServerGS：全局的一些信息，会话计数、task/worker进程池；
1. swListenPort：监听的端口列表；
1. swConnection：每一个客户端连接会对应一个swConnection；
1. swSession：会话id与套接字描述符的对应关系，每一个客户端连接会对应一个swSession，其实这个具体作用我还看不出来= =。

以如下服务端代码为例：
{% highlight php linenos %}
{% raw %}
<?php
$serv = new swoole_server("127.0.0.1", 9502);
$serv->set(array(
    'worker_num' => 4,   //工作进程数量
    'daemonize' => false, //是否作为守护进程
));
$serv->on('connect', function ($serv, $fd){
    echo "Client:Connect.\n";
});
$serv->on('receive', function ($serv, $fd, $from_id, $data) {
        //sleep(2);//sleep 20 seconds
    echo "receive : $data \n";
    $serv->send($fd, 'From TcpServer Swoole: '.$data);
    $serv->close($fd);
});
$serv->on('close', function ($serv, $fd) {
    echo "Client: Close.\n";
});
$serv->start();
{% endraw %}
{% endhighlight %}

上述代码执行时，swoole\_server启动的函数调用流程图如下：
![](/assets/img/201604160102.png)

在master进程、thread线程、worker进程中都有各自的reactor进行事件轮询，并且各自向reactor注册了一些事件。

master：

{% highlight c linenos %}
{% raw %}
//注册的handle，swoole扩展了基本的事件类型
SW_FD_LISTEN : swServer_master_onAccept//当SW_FD_LISTEN事件发生时，会回调swServer_master_onAccept函数

//监听的文件描述符
main_reactor_ptr->add(main_reactor_ptr, ls->sock, SW_FD_LISTEN)//监听ls->sock发生的SW_FD_LISTEN事件
{% endraw %}
{% endhighlight %}

thread：

{% highlight c linenos %}
{% raw %}
//注册的handle
SW_FD_CLOSE : swReactorThread_onClose
SW_FD_PIPE | SW_EVENT_READ : swReactorThread_onPipeReceive
SW_FD_PIPE | SW_EVENT_WRITE : swReactorThread_onPipeWrite

SW_FD_UDP : swReactorThread_onPackage
SW_FD_TCP | SW_EVENT_WRITE : swReactorThread_onWrite
SW_FD_TCP | SW_EVENT_READ : swReactorThread_onRead

//监听的文件描述符
add(reactor, pipe_fd, SW_FD_PIPE)//pipe_fd是与worker进程数据交互的管道
sub_reactor->add(sub_reactor, new_fd, SW_FD_TCP | SW_EVENT_WRITE)//由master添加到thread的reactor中，new_fd是accept的客户端连接
{% endraw %}
{% endhighlight %}

worker：

{% highlight c linenos %}
{% raw %}
//注册的handle
SW_FD_PIPE : swWorker_onPipeReceive
SW_FD_PIPE | SW_FD_WRITE : swReactor_onWrite

//监听的文件描述符
add(SwooleG.main_reactor, pipe_worker, SW_FD_PIPE | SW_EVENT_READ)//pipe_worker是与reactor thread数据交互的管道
{% endraw %}
{% endhighlight %}

以如下客户端代码为例：
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

var_dump($client->connect('127.0.0.1', 9502, 0.5));
{% endraw %}
{% endhighlight %}

当客户端发起请求时，服务端处理流程如下：
![](/assets/img/201604160103.png)