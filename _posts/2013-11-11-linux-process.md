---
layout: post
title: "Linux 进程、进程组、会话周期、控制终端"
description: ""
category: Linux
tags: [系统]
---
{% include JB/setup %}

### 进程

> 为使程序能并发执行，且为了对并发（并发指两个或多个事件在同一时间间隔内发生）执行的程序加以描述和控制，于是引入了“进程”的概念。

### 守护进程

> 守护进程，也就是通常说的Daemon进程，是Linux中的后台服务进程。它是一个生存期较长的进程，通常独立于控制终端并且周期性地执行某种任务或等待处理某些发生的事件。守护进程是脱离于终端并且在后台运行的进程。守护进程脱离于终端是为了避免进程在执行过程中的信息在任何终端上显示并且进程也不会被任何终端所产生的终端信息所打断。

### 进程组

> 每个进程都属于一个进程组。每个进程组都有一个组长进程，组长进程的进程号等于进程组ID。只要某个进程组中有一个进程存在，则该进程组就存在，与组长进程是否终止无关。从进程组创建开始到其中最后一个进程离开为止的时间区间成为进程组的生存期。进程组中最后一个进程可以终止或者转移到另一个进程组中。

### 会话周期

> 会话周期是一个或多个进程组的集合。通常，一个会话开始于用户登录，终止于用户退出，在此期间该用户运行的所有进程都属于这个会话期。

### 控制终端

>  会话的领头进程打开一个终端之后, 该终端就成为该会话的控制终端 (SVR4/Linux) 与控制终端建立连接的会话领头进程称为控制进程 (session leader) 。一个会话只能有一个控制终端 ，产生在控制终端上的输入和信号将发送给会话的前台进程组中的所有进程 ，终端上的连接断开时 (比如网络断开或 Modem 断开), 挂起信号将发送到控制进程(session leader) 。平时在X-window下是使用的terminal称为伪终端，但它也是一个控制终端。

### 僵死进程

> 在`fork()/execve()`过程中，假设子进程结束时父进程仍存在，而父进程`fork()`之前既没安装`SIGCHLD`信号处理函数调用`waitpid()`等待子进程结束，又没有显式忽略该信号，则子进程成为僵死进程，无法正常结束，此时即使是root身份`kill -9`也不能杀死僵死进程。补救办法是杀死僵尸进程的父进程(僵死进程的父进程必然存在)，僵死进程成为"孤儿进程"，过继给1号进程`init`，`init`始终会负责清理僵死进程。

#### 守护进程编写的要点：

1、在父进程（此时是一个进程组的组长）中使用fork()产生子进程（将来的守护进程由它产生）；

2、调用setsid()，用于生成一个新的会话
注意如果当前进程是会话组长时，调用失败。第一点已经可以保证进程不是会话组长了，所以setsid()调用成功后，进程成为新的会话组长和新的进程组长，并与原来的登录会话和进程组脱离。由于会话对控制终端的独占性，进程同时与控制终端脱离；

3、禁止进程重新打开控制终端
第二步之后，进程已经成为无终端的会话组长。但它可以重新申请打开一个控制终端。可以通过使进程不再成为会话组长来禁止进程重新打开控制终端，在上面的控制终端中已经提到了只有会话组长才能打开控制终端；

4、关闭打开的文件描述符
进程从创建它的父进程那里继承了打开的文件描述符。如不关闭，将会浪费系统资源，造成进程所在的文件系统无法卸下以及引起无法预料的错误；

5、改变当前工作目录 
进程活动时，其工作目录所在的文件系统不能卸下。一般需要将工作目录改变到根目录。对于需要转储核心，写运行日志的进程将工作目录改变到特定目录如`/tmp`；

6、重设文件创建掩模 
进程从创建它的父进程那里继承了文件创建掩模。它可能修改守护进程所创建的文件的存取位。为防止这一点，将文件创建掩模清除：`umask(0)`；

7、处理SIGCHLD信号 
处理SIGCHLD信号并不是必须的。但对于某些进程，特别是服务器进程往往在请求到来时生成子进程处理请求。如果父进程不等待子进程结束，子进程将成为僵尸进程（zombie）从而占用系统资源。如果父进程等待子进程结束，将增加父进程的负担，影响服务器进程的并发性能。在Linux下可以简单地将`SIGCHLD`信号的操作设为`SIG_IGN`。 
`signal(SIGCHLD,SIG_IGN); `
这样，内核在子进程结束时不会产生僵尸进程。这一点与BSD4不同，BSD4下必须显式等待子进程结束才能释放僵尸进程。 

8、重定向标准输入、标准输出和标准错误输出到`/dev/null`，防止发生EIO错误。

先来看看上面第1步是为何，编译执行下面代码：
{% highlight cpp linenos %}
#include<sys/types.h>
#include<unistd.h>
#include<stdio.h>
#include<stdlib.h>
#include<errno.h>

int main() {
    pid_t pid;
    pid = fork();

    if (pid == -1) {
        perror("fork");
    } else if (pid > 0) {//父进程
        printf("Parent pid:%d\n", getpid());
        sleep(30);
        exit(0);
    } else {
        printf("Child pid:%d\n", getpid());
        sleep(30);//子进程
        exit(0);
    }
}
{% endhighlight %}
然后在另一个terminal中执行：`ps axj | grep tp`

![process](/assets/img/201311130101.png)

从上图可以看出父进程的PID是7667，且是进程组7667的组长，所以我们写守护进程的第一步就是要使用`fork()`产生子进程。

### 守护进程实例

源代码：

{% highlight cpp linenos %}
#include<sys/types.h>
#include<sys/param.h>
#include<sys/stat.h>
#include<fcntl.h>
#include<unistd.h>
#include<stdio.h>
#include<stdlib.h>
#include<errno.h>
#include<time.h>

void init_daemon() {
    pid_t pid;
    int i;

    if (pid = fork()) {//是父进程，结束父进程
        exit(0);
    } else if (pid < 0) {//fork()失败
        perror("fork");
    }

    //是第一子进程，继续执行
    setsid();//第一子进程成为新的会话组长和进程组长，并与控制终端分离，为了防止其再打开一个控制终端，再调用一次fork()
    if (pid = fork()) {//是第一子进程，结束第一子进程
        exit(0);
    } else if (pid < 0) {//fork()失败
        exit(1);
    }

    //是第二子进程，继续执行，已经不再是会话组长，执行关闭文件描述符操作
    for (i = 0; i < NOFILE; i++) {
        close(i);
    }

    //改变工作目录
    chdir("/tmp");

    //重设文件创建掩模
    umask(0);

    //重定向标准输入、标准输出、标准错误输出到/dev/NULL
    /*
     * STDERR_FILENO = 2 标准错误输出 
     * STDIN_FILENO = 0 标准输入 
     * STDOUT_FILENO = 1 标准输出
     */
    int fp = open("/dev/null", O_RDWR);
    dup2(fp, STDERR_FILENO);
    dup2(fp, STDIN_FILENO);
    dup2(fp, STDOUT_FILENO);

    //处理SIGCHLD信号
    signal(SIGCHLD,SIG_IGN);
    return;
}

int main() {
    FILE *fp;
    time_t t;
    init_daemon();

    while (1) {
        sleep(10);//睡眠10妙
        fp = fopen("test.log", "a");
        if (fp != NULL) {
            time(&t);
            fprintf(fp, "Now time is %s\r\n", asctime(gmtime(&t)));
            fclose(fp);
        }
    }
}
{% endhighlight %}

查看守护进程，执行`ps axj | grep tp`：

![process](/assets/img/201311130102.png)

由上图可以看出，守护进程的PID为8368，所属进程组为8367，会话ID为8367，没有控制终端，最重要的一点是父进程为1（即init进程）。