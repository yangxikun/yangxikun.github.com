---
layout: post
title: "ctrl+c'杀不死'docker container"
description: ""
category: Docker
tags: []
---
{% include JB/setup %}

有时使用`ctrl+c`杀掉容器时，发现行不通，得用`docker stop <container>`才行，于是就像弄明白为什么。

从dockerfile官方文档：[https://docs.docker.com/engine/reference/builder/#entrypoint](https://docs.docker.com/engine/reference/builder/#entrypoint)，得知`ENTRYPOINT`有两种形式：

1. `ENTRYPOINT ["executable", "param1", "param2"]` （exec格式，推荐使用）
1. `ENTRYPOINT command param1 param2` （shell格式）

使用exec格式，容器的init进程（即PID为1的进程）就是我们指定的可执行程序。

使用shell格式，容器的init进程是`/bin/sh`，我们的`ENTRYPOINT`是以`/bin/sh -c`的子进程启动的。

<!--more-->

我基于`centos:latest`构建了两个镜像，Dockerfile如下：

```plaintext
FROM centos:latest
ENTRYPOINT sleep 1000
```

```plaintext
FROM centos:latest
ENTRYPOINT ["sleep", "1000"]
```

分别启动了交互式的容器：

```plaintext
CONTAINER ID        IMAGE                         COMMAND                  CREATED             STATUS              PORTS               NAMES
30cc518f3d26        yangxikun/docker-test:shell   "/bin/sh -c 'sleep 1…"   8 seconds ago       Up 6 seconds                            cranky_yalow
f303e8705113        yangxikun/docker-test:exec    "sleep 1000"             17 seconds ago      Up 15 seconds                           suspicious_banach
```

都按下`ctrl+c`，发现只会在终端回显`^C`，进入容器中查看：

![](/assets/img/201802160101.png)

发现PID为1的进程都是`sleep 1000`，这不对呀，容器cranky_yalow的PID为1的进程应该是`/bin/sh`。

尝试在宿主机ubuntu执行`/bin/sh -c "sleep 1000"`，确实是fork了子进程执行`sleep 1000`。于是怀疑是`/bin/sh`的实现上，centos有所不同，后来Google了一番，才发现`/bin/sh`其实是一个软连接，使用`readlink -e $(which sh)`可以找出对应的程序，[What is the sh -c command?](https://askubuntu.com/questions/831847/what-is-the-sh-c-command)。

在ubuntu下：`/bin/sh -> dash`，在centos下：`/bin/sh -> bash`。

在ubuntu下执行`bash -c "sleep 1000"`，确实没有fork子进程。

为啥`ctrl+c`无效呢，进入到容器里，直接`kill -9 1`也无效。Google一番，找到了答案：[https://www.quora.com/Is-it-possible-to-kill-the-init-process-in-Linux-by-the-kill-9-command](https://www.quora.com/Is-it-possible-to-kill-the-init-process-in-Linux-by-the-kill-9-command)。

`man 2 kill` 手册note说明：

> The only signals that can be sent to process ID 1, the init process, are those for which init has explicitly installed signal handlers. This is done to assure the system is not brought down accidentally.

也就是说只有PID为1的进程显示设置了某个信号的处理函数，该信号才会被发送到PID为1的进程。

所以可以猜想之前实验的ENTRYPOINT指定的sleep进程对SIGINT和SIGTERM信号没有进行捕获或忽略了（SIGKILL信号无法捕获或忽略）。

将基础镜像从`centos:latest`改为`ubuntu:latest`，从新启动容器，进入容器查看PID为1的进程：

![](/assets/img/201802160102.png)

`ctrl+c`为何能够kill掉容器compassionate_ride（即shell格式）？是dash捕获到SIGINT之后退出吗？不一定，因为分配了tty，dash fork出来的sleep子进程同样也能收到SIGINT，也可能是sleep子进程的退出，使得dash也退出了。

在我的ubuntu宿主机做个测试，执行`sh -c "sleep 1000"`，可以看到产生了如下两个进程：

```plaintext
rokety   18371  0.0  0.0   4596   928 pts/4    S+   20:57   0:00 sh -c sleep 1000
rokety   18372  0.0  0.0   9148   708 pts/4    S+   20:57   0:00 sleep 1000
```

发送SIGINT给进程18371：`kill -SIGINT 18371`，进程没有退出，说明dash捕获了SIGINT，但没有退出。

发送SIGINT给进程18372：`kill -SIGINT 18372`，进程都退出了，说明sleep在接收到SIGINT信号后退出了，dash也跟着退出。

那么`docker stop`又是怎么将容器kill掉的？查看其官方文档[https://docs.docker.com/engine/reference/commandline/stop/#usage](https://docs.docker.com/engine/reference/commandline/stop/#usage)，`docker stop`首先会向容器主进程（即init进程）发送SIGTERM信号，等待一段时间（默认10s），如果容器还没停止，就发送SIGKILL信号。

可在上文中，我们已经提到，如果容器init进程没有显示设置对信号的处理或者说忽略信号，SIGTERM信号是不起作用的，即使在容器里执行SIGKILL也无效。

那为何`docker stop`能起作用呢？容器实际上是一个运行在宿主机上的进程，在宿主机上，同样可以通过kill发送信号给对应的容器init进程，这时候容器init进程的PID就不是1了，是否就不会受到kill机制的保护了，做个实验看看：

先启动一个exec格式的容器，通过`ps aux | grep sleep`，可以在宿主机上查看到对应的容器进程：

```plaintext
root     21628  1.4  0.0   4376   744 pts/0    Ss+  21:08   0:00 sleep 1000
```

发送SIGINT信号：`sudo kill -SIGINT 21628`，发现容器没有退出，说明还是受到了保护。发送SIGKILL信号试试，发现容器退出了（注意，在容器里发送SIGKILL给PID 1进程是无效的）。

#### 总结
- - -

**ENTRYPOINT exec格式（推荐）**

* 可以配合CMD指令，CMD指令的设置可以作为ENTRYPOINT指令的默认参数
* CMD指令设置的参数可以被docker run指定的参数覆盖
* `ENTRYPOINT ["executable", "$env_param1"]`，环境变量不会发生替换，除非包多一层shell `ENTRYPOINT ["sh", "-c", "echo $FOO"]`
* 能够接收到docker stop的SIGTERM信号

**ENTRYPOINT shell格式**

* CMD指令不起作用
* 环境变量会发生替换
* ENTRYPOINT 会作为`/bin/sh -c`的子程序启动，无法接收到docker stop的SIGTERM信号

对容器init进程，没有显示声明要捕获的信号，不会被发送到，可以在宿主机上发送SIGKILL信号，将其kill（容器内不行），这也是docker stop最终能停止容器的原因。

#### 参考资料
- - -
* [Docker and the PID 1 zombie reaping problem](https://blog.phusion.nl/2015/01/20/docker-and-the-pid-1-zombie-reaping-problem/)
* [How can I check what signals a process is listening to?](https://unix.stackexchange.com/questions/85364/how-can-i-check-what-signals-a-process-is-listening-to)