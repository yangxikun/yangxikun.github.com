---
layout: post
title: "golang 汇编学习小结"
description: ""
category: GoLang
tags: []
---
{% include JB/setup %}

本文学习参考自：[plan9 assembly 完全解析](https://github.com/cch123/golang-notes/blob/master/assembly.md) 和 [Go汇编语言](https://chai2010.cn/advanced-go-programming-book/ch3-asm/readme.html)

#### 寄存器

Go 汇编的4个伪寄存器，手写Go 汇编的时候使用：

* FP：用于引用参数和返回值
* PC：程序计数器
* SB：用于引用全局符号
* SP：指向当前栈帧的BP，引用局部变量

<!--more-->

#### 栈结构

函数的定义：TEXT symbol(SB), [flags,] $framesize[-argsize]

```plaintext
                                       caller
                                 +------------------+
                                 |                  |
       +---------------------->  --------------------
       |                         |                  |
       |                         | caller parent BP |
       |           BP(pseudo SP) --------------------
       |                         |                  |
       |                         |   Local Var0     |
       |                         |------------------|
       |                         |                  |
       |                         |   .......        |
       |                         |------------------|
       |                         |                  |
       |                         |   Local VarN     |
       |                         --------------------
       |                         |                  |
       |                         |   return val1    |
       |                         |------------------|
       |                         |                  |
       |                         |   return val0    |
       |                         --------------------
 caller stack frame              |                  |
                                 |   callee arg2    |
       |                         |------------------|
       |                         |                  |
       |                         |   callee arg1    |
       |                         |------------------|
       |                         |                  |
       |                         |   callee arg0    |
       |                         ----------------------------------------------+   FP(virtual register)
       |                         |                  |                          |
       |                         |   return addr    |  parent return address   |
       +---------------------->  +------------------+---------------------------    <-------------------------------+
                                                    |  caller BP               |                                    |
                                                    |  (caller frame pointer)  |                                    |
                                     BP(pseudo SP)  ----------------------------                                    |
                                                    |                          |                                    |
                                                    |     Local Var0           |                                    |
                                                    ----------------------------                                    |
                                                    |                          |
                                                    |     Local Var1           |
                                                    ----------------------------                            callee stack frame
                                                    |                          |
                                                    |       .....              |
                                                    ----------------------------                                    |
                                                    |                          |                                    |
                                                    |     Local VarN           |                                    |
                                  SP(Real Register) ----------------------------                                    |
                                                    |                          |                                    |
                                                    |                          |                                    |
                                                    |                          |                                    |
                                                    |                          |                                    |
                                                    |                          |                                    |
                                                    +--------------------------+    <-------------------------------+

                                                              callee
```

* `return addr`是执行CALL指令时，将过程调用返回地址入栈的，同时物理寄存器SP+8字节
* 当过程调用的frame size > 0时，`caller BP`才会入栈
* 通过`go tool compile -S foo.go`或`go tool objdump bin`显示的汇编里没有伪寄存器