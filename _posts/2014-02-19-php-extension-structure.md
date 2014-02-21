---
layout: post
title: "PHP扩展结构"
description: ""
category: PHP
tags: [PHP扩展]
---
{% include JB/setup %}

#### 必要的文件

1. extname.c
2. php_extname.h
3. config.m4
4. config.w32

#### php_extname.h

1. 一些必要的宏定义
2. 函数声明，注意自己写的提供给脚本使用的函数也应该在这里进行声明
3. 全局变量定义

<!--more-->
#### extname.c文件结构

1. 文件头：许可声明、作者……
2. include包含一些必要文件
3. 函数参数说明
4. 函数入口
5. 模块入口
6. php.ini配置入口
7. PHP_MINIT_FUNCTION
8. PHP_MSHUTDOWN_FUNCTION
9. PHP_RINIT_FUNCTION
10. PHP_RSHUTDOWN_FUNCTION
11. PHP_MINFO_FUNCTION
12. PHP_FUNCTION
13. 页脚注释

#### 全局变量的声明

手册：[Introduction to globals in a PHP extension ](http://www.php.net/manual/en/internals2.structure.globals.php)