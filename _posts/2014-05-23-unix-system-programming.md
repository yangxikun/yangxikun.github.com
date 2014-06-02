---
layout: post
title: "《UNIX Systems Programming》学习笔记一"
description: ""
category: linux
tags: [linux系统编程]
---
{% include JB/setup %}

#### 缓冲区溢出攻击
- - -

[缓冲区溢出攻击](http://baike.baidu.com/view/700134.htm#6)

#### 进程地址空间布局
- - -

一个程序镜像在内存中的简单布局：
![program image](/assets/img/201405230101.png)

初始化的静态变量在编译的时候将会为其分配相应的地址空间并初始化，所以编译后的程序较大，如下两个程序：

程序`largearrayinit.c`：
{% highlight c %}
#include <stdio.h>

int arr[50000] = {1, 2, 3, 4};
int main(int argc, char *argv[])
{
    arr[0] = 5;
    return 0;
}
{% endhighlight %}

程序`largearray.c`：
{% highlight c %}
#include <stdio.h>

int arr[50000];
int main(int argc, char *argv[])
{
    arr[0] = 5;
    return 0;
}
{% endhighlight %}

编译后的结果：
![largearray](/assets/img/201405230102.png)

#### 一些来自库函数的编程准则
- - -

1. 使用返回值来传递信息，并且错误信息能够被调用程序简单地获取，例如使用`errno`；
1. 不要在函数调用内执行`exit`，取而代之的是返回一个错误的值（-1），并且允许调用程序灵活的处理错误；
1. 不要对`buffers`作出任何假设；
1. 当需要使用限制值时，使用系统定义的常量值`limits.h`；
1. 不要重复造不必要的轮子，尽量使用标准库函数；
1. 不要试图修改输入参数，除非真的必要；
1. 尽量不使用静态变量和动态内存分配，如果自动变量能工作得好的话；
1. 分析所有调用`malloc`家族系统调用的地方，确保在适当的时候释放申请的内存；
1. 分析一切可能的来自信号的中断；
1. 仔细思考整个程序是如何`terminates`。

#### 线程安全函数
- - -

`POSIX`定义了线程安全函数，以`_r`结尾，例如`strtok_r`。

<!--more-->

#### 进程终止
- - -

当一个进程终止时，操作系统回收分配给该进程的资源，更新统计信息，以及提醒其他进程（使用了`wait`系统调用）。

在UNIX中，一个进程不会在终止的是否完全地释放资源，直到它的父进程等待它。如果父进程没有等待子进程终止，那么终止的子进程将会成为`zombie`进程。一个`zombie`进程是一个不活动的进程，当父进程等待它时，它的资源才会被完全释放。

当一个进程终止时，它的孤儿进程（即父进程先终止，而子进程未终止）以及`zombie`进程将会由一个特殊的进程`init`接管。`init`是一个PID为1的进程，它会周期性的等待子进程。

进程正常的终止：
* 从`main`函数返回；
* 调用`exit, _Exit, _exit`。

`_Exit`和`_exit`作用相同，与`exit`的区别是：`exit`会调用通过`atexit(), on_exit()`注册的函数，而另外两个不会。

进程非正常的终止（可能会产生核心转储，用户注册的exit函数将不会被调用）：
* 调用`abort()`；
* 一个使进程终止的信号，例如`Ctrl-C`；
* 一个内部错误，例如试图访问非法的内存空间，段错误。

#### 文件描述符
- - -

当执行如下语句`myfd = open("/home/rokety/me.txt", O_RONLY);`：

![file descriptor](/assets/img/201405240101.png)

用户空间：保存了进程打开的文件描述符表。

内核空间：保存了系统文件表（表中的每一项指向了文件的入口，记录了读写指针位移、读写模式等信息）和节点表（文件的入口）。

系统文件表和in-memory inode表每一项都会有一个计数值，记录有多少引用，当引用减少为0时，就从表项中删除。例如：当执行`close(myfd)`时，会先从文件描述符表中删除一项，然后将系统文件表和in-memory inode表中被引用的项count减1，当count为0时，即可以删除。

#### 文件指针和缓冲区
- - -

执行如下代码：
{% highlight c %}
FILE *myfp;

if ((myfp = fopen("/home/rokety/me.txt", "w")) == NULL) {
    perror("Failed to open /home/rokety/me.txt");
} else {
    fprintf(myfp, "Hello rokety~");
}
{% endhighlight %}

![file pointers](/assets/img/201405240102.png)

当向文件指针写入数据时，其实是写入了一个FILE结构体的buffer中，当buffer满时，将会调用write将数据写入磁盘。可以通过`setvbuf`取消缓冲区，或者`fflush`强制刷出缓冲区内容。

#### 重定向
- - -

代码执行流程：

1. 打开`my.file`文件；
1. 使用`dup2`系统调用，将标准输出重定向到`my.file`；
1. 关闭`my.file`文件描述符。

文件描述符表变化：

![redirection](/assets/img/201406020101.png)

#### 硬链接和软链接
- - -

![hard symbolic](/assets/img/201406020102.png)
