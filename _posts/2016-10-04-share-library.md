---
layout: post
title: "静态库与动态库"
description: "静态库与动态库"
category: C
tags: []
---
{% include JB/setup %}

学习自：《程序员的自我修养》和[Static, Shared Dynamic and Loadable Linux Libraries](http://www.yolinux.com/TUTORIALS/LibraryArchives-StaticAndDynamic.html)

在Linux下有两种类型的C/C++库：

静态库：以lib{name}.a命名，会被链接为可执行文件的一部分。

被动态链接的共享库（以下简称动态库）：以lib{name}.so命名，可执行文件运行时才进行链接，称为动态链接。链接的方式有：

> * 动态链接器在可执行文件真正开始运行前，链接动态库；
* 动态连接器在可执行文件运行过程中，链接动态库，又称为延迟绑定，是为了避免程序在启动时就链接了所有依赖的库，等到真正需要的时候才进行链接；
* 可执行文件运行过程中，调用`dlopen()`系统调用链接动态库，又称为运行时装载，PHP的扩展就是通过该方式加载的。

<!--more-->

### 生成静态库
- - -

测试文件：

```c
//echo.c
#include <stdio.h>

void echo()
{
    printf("Hello World!\n");
}
```

```c
//echo.h
void echo();
```

```c
//main.c
#include "echo.h"

int main(int argc, char *argv[])
{
    echo();
    return 0;
}
```

编译libecho.a静态库：`gcc -c echo.c -o echo.o;ar -qcv libecho.a echo.o`，ar是一个用于将目标文件打包为静态库的工具。可通过`nm -s libecho.a`查看打包后的内容。

编译可执行程序：`gcc -o main main.c -Wall -L. -lecho`。

> * -L：链接时的搜索目录，优先在该目录进行搜索
* -l：需要的库

这里，如果编译可执行程序时，命令写成：`gcc -Wall -L. -lecho -o main main.c`的话，会报错：`undefined reference to echo`。原因是链接器根据命令行提供的目标文件顺序，从左到右搜索未决符号，并做标记，直到某个目标文件能够对符号进行决议。所以，如果把main.c放在-lecho之后的话，就会存在未决符号。详情可查看：[Why does the order in which libraries are linked sometimes cause errors in GCC?](http://stackoverflow.com/questions/45135/why-does-the-order-in-which-libraries-are-linked-sometimes-cause-errors-in-gcc)

### 生成动态库
- - -

测试文件相同。

编译libecho.so动态库：`gcc -shared -fPIC -Wl,-soname,libecho.so.1 -o libecho.so.1.0 echo.c`

> * -shared：产生一个可被链接到可执行文件中的共享目标文件
* -fPIC：产生地址无关代码，动态库所要求的一个特性，为了不同可执行程序能够共享同一份动态库的代码段，以达到节省内存的目的。具体可查阅《程序员的自我修养》P188。
* -Wl,-soname,libecho.so.1：传递给链接器的参数，设置输出的动态库名称为libcho.so.1
* -o：动态库文件名

编译可执行程序：`gcc -o main main.c -Wall -L. -lecho`，会报错：`cannot find -lecho`，需要做个软链接：`ln -s libecho.so.1.0 libecho.so`，因为这里，链接器默认搜索的是libecho.so目标文件。

运行`./main`，又报错：`error while loading shared libraries: libecho.so.1: cannot open shared object file: No such file or directory`，因为这里，链接器搜索的是libecho.so.1目标文件，需要再加一个软链接：`ln -s libecho.so.1.0 libecho.so.1`，并且设置环境变量：`export LD_LIBRARY_PATH=.`。

通过ldd命令可查看可执行文件依赖的动态库`ldd main`：

```c
linux-vdso.so.1 (0x00007fff04fd8000)
libecho.so.1 => ./libecho.so.1 (0x00007fd3f5e21000)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fd3f5a76000)
/lib64/ld-linux-x86-64.so.2 (0x00007fd3f6022000)
```

### Linux下库的路径与库的版本
- - -

动态库命名规则：libname.so.x(主版本号).y(次版本号).z(发布版本号)。

* 不同主版本号的库之间不兼容
* 主版本号相同，次版本号不同，高的次版本号的库向后兼容
* 主版本号和次版本号都相同，发布版本号不同的库之间完全兼容

SO-NAME：

用于说明程序依赖于哪些版本的哪些动态库。Linux中采用SO-NAME的命名机制来记录动态库的依赖关系。每个动态库都有一个对应的“SO-NAME”，这个SO-NAME即动态库的文件名去掉次版本号和发布版本号，保留主版本号。例如一个动态库叫做libfoo.so.2.6.1，那么它的SO-NAME为libfoo.so.2。

在Linux系统中，系统会为每个动态库在它所在的目录创建一个跟“SO-NAME”相同的并且指向它的软链接。如果某个动态库存在多个主版本号相同，而次版本号或发布版本号不同，那么会软链接到次版本号和发布版本号最新的那个动态库。

共享库（静态库+动态库）系统路径：

* /lib、/usr/lib：常用的、成熟的，一般是系统本身所需要的库
* /usr/local/lib：非系统所需要的第三方库，例如异步redis客户端：libhiredis.so

共享库搜索：

先在LD_LIBRARY_PATH中查找，再在/lib、/usr/lib和由/etc/ld.so.conf配置文件指定的目录中查找共享库。为何没有/usr/local/lib，因为/usr/local/lib是在/etc/ld.so.conf配置文件指定的（以我的debian为例，在/etc/ld.so.conf.d/libc.conf指定了/usr/local/lib）。
