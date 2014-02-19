---
layout: post
title: "GoLang 基础笔记"
description: ""
category: GoLang
tags: [GoLang基础]
---
{% include JB/setup %}

#### 字符串

1. 使用"和\`定义，类型：string；
2. 无法修改（类似C语言中的字符串常量`char *str="Hello"`）；
3. 可进行切片操作，例如：`str[startIndex:endIndex+1]`。

<!--more-->
#### slice

1. 引用类型；
2. 可进行切片操作，例如：`aSlice[startIndex:endIndex+1]`。
3. `len()`：获取长度
4. `cap()`：获取最大容量；
5. `append()`：向slice追加一个或者多个元素，然后返回一个和slice一样类型的slice，如果超过了`cap`，那么会动态分配新的数组空间，与原来的数组空间分离了；
6. `copy()`：从源slice的src中复制元素到目标dst，并且返回复制的元素个数。

### map

1. 引用类型
2. 元素无序性，每次打印出来看到的顺序可能互不相同；

### 类型转换

1. `[]byte`和`string`可相互转换；
2. `interface{}`可以使用.(type)进行类型转换。