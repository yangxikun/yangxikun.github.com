---
layout: post
title: "C开发时遇到一些问题"
description: ""
category: C
tags: []
---
{% include JB/setup %}

1、局部变量的地址赋值给了全局变量指针，导致程序出现诡异的Segmentation fault，且是在MAC下，Linux下碰巧没遇到！。

排查问题：

当时发现全局变量`*executor_globals.return_value_ptr_ptr`的值在经过某个函数调用的时候发生了改变（至于怎么发现这点的忘了= =）。

如下图，在经过`efree(client->cli)`之后，`*executor_globals.return_value_ptr_ptr`的值从`0x000000010a244318`变为了`0x000000010a244340`。

![](/assets/img/201609250101.png)

<!--more-->

当时就想efree函数不应该会修改`*executor_globals.return_value_ptr_ptr`的值，于是使用gdb单步调试到`efree(client->cli)`之前，然后设置watchpoint：`watch *executor_globals.return_value_ptr_ptr`。接着执行`c`，发现在如下图位置break了：

![](/assets/img/201609250102.png)

这里只对局部变量p做了修改，为何会影响到`*executor_globals.return_value_ptr_ptr`的值？于是打印出p变量的地址，发现与`executor_globals.return_value_ptr_ptr`的值是相同的。

为何`executor_globals.return_value_ptr_ptr`的值为一个堆栈变量的地址呢？经过一番查找发现原来是在某一处函数调用中把某个堆栈变量的地址赋值给了`executor_globals.return_value_ptr_ptr`。

2、gcc4.4下的编译告警导致诡异的Segmentation fault

发现程序在gcc4.4下编译后运行会出现诡异的Segmentation fault，一开始一直没搞明白为什么，后来发现gcc4.4(开启了-O2)编译的时候比gcc4.8(没有开启-O2)编译的时候多了一些warning：

* `dereferencing pointer ‘v.327’ does break strict-aliasing rules`
* `dereferencing type-punned pointer will break strict-aliasing rules`

经查找，这些warning产生的原因是：[Strict Aliasing，神坑？](http://blog.kongfy.com/2015/09/strict-aliasing%EF%BC%8C%E7%A5%9E%E5%9D%91%EF%BC%9F/)。于是编辑Makefile加上`-fno-strict-aliasing`，重新编译之后就正常了。

3、unsigned类型的算术运算表达式，计算结果溢出，最终导致程序出现Segmentation fault

bug代码：

```c
//n_buf为uint32_t类型
if (n_buf - 4 < client->response.packet_length)
{
  return SW_ERR;
}
```

当n_buf < 4时，`n_buf - 4`的值转换为uint32_t类型就是一个较大的正数了，这样就没法走return的逻辑了。

fix后的代码：

```c
if ((n_buf < 4) || (n_buf - 4 < client->response.packet_length))
{
  return SW_ERR;
}
```

4、waitpid返回no child process错误

父进程在终止时向子进程发送SIGTERM信号，然后调用waitpid等待子进程退出，发现waitpid有时返回了no child process的错误信息。经排查发现父进程设置了SIGCHILD信号的处理函数，并在处理函数中调用了waitpid。

解决该问题的方法，在kill之前，使用sigprocmask对SIGCHILD信号进行阻塞。