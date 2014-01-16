---
layout: post
title: "Linux线程实现"
description: ""
category: Linux
tags: [线程]
---
{% include JB/setup %}

*学习自：[linux进程与线程](http://wenx05124561.blog.163.com/blog/static/1240008052011717114011994/)*

### 进程

> 进程是程序运行的实体，并且占有一定的系统资源。

### 线程

> Linux的进程，线程实现是在核外进行的，核内提供的是创建进程的接口`do_fork()`。内核提供了 三个系统调用`__clone()`和`fork()`以及`vfort()`，最终都用不同的参数调用`do_fork()`核内API。 `do_fork()` 提供了很多参数，包括`CLONE_VM`（共享内存空间）、`CLONE_FS`（共享文件系统信息）、`CLONE_FILES`（共享文件描述符表）、 `CLONE_SIGHAND`（共享信号句柄表）和`CLONE_PID`（共享进程ID，仅对核内进程，即0号进程有效）等。

<!--more-->
> Linux下不管是多线程编程还是多进程编程，linux通过`clone()`系统调用实现产生进程或者线程。无论是`fork()`，还是`vfork()`、`__clone()`最后都根据各自需要的参数标志去调用`clone()`，然后有`clone()`去调用`do_fork()`。这样一说，最终都是用`do_fork()`实现的多进程编程，只是进程创建时的参数不同，从而导致有不同的共享环境。

> 当使用`pthread_create()`来创建线程时,则最终通过设置参数来调用`__clone()`，而这些参数又全部传给核内的`do_fork()`，从而创建的"进程"拥有共享的运行环境，只有栈是独立的，由 `__clone()`传入。我们说在linux中，线程仅仅是一个使用共享资源的轻量级进程。调用`clone()`时候`flag`指定为`CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND`。

*错误检查：以前学过的系统函数都是成功返回0，失败返回-1，而错误号保存在全局变量`errno`中，而pthread库的函数都是通过返回值返回错误号，虽然每个线程也都有一个`errno`，但这是为了兼容其它函数接口而提供的，`pthread`库本身并不使用它，通过返回值返回错误码更加清晰。由于`pthread_create`的错误码不保存在errno中，因此不能直接用`perror(3)`打印错误信息，可以先用`strerror(3)`把错误码转换成错误信息再打印。*