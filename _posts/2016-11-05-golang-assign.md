---
layout: post
title: "golang中的赋值"
description: "golang中的赋值"
category: GoLang
tags: [GoLang基础]
---
{% include JB/setup %}

先看看文档中关于赋值的说明[Assignability](https://golang.org/ref/spec#Assignability)：

A value x is assignable to a variable of type T ("x is assignable to T") in any of these cases:

1. x's type is identical to T.
1. x's type V and T have identical underlying types and at least one of V or T is not a named type.
1. T is an interface type and x implements T.
1. x is a bidirectional channel value, T is a channel type, x's type V and T have identical element types, and at least one of V or T is not a named type.
1. x is the predeclared identifier nil and T is a pointer, function, slice, map, channel, or interface type.
1. x is an untyped constant representable by a value of type T.

<!--more-->

逐条规则进行理解：

1、x的类型与T相同

什么样的类型在golang中是相同的呢？文档中的说明[Type identity](https://golang.org/ref/spec#Type_identity)：

两个类型，要么是相同的，要么是不同的。

* 两个命名类型（named types）是相同的，仅当他们的类型名称（type name）是从同一个TypeSpec产生的。
* 命名类型与非命名类型（unnamed type）总是不相同的。
* 两个非命名类型是相同的，仅当他们的类型字面量（type literal）是相同的。

上面3条规则中的关键词得弄明白：

**TypeSpec**

文档中的说明：[Type declarations](https://golang.org/ref/spec#Type_declarations)

类型声明语法：

```
TypeDecl     = "type" ( TypeSpec | "(" { TypeSpec ";" } ")" ) .
TypeSpec     = identifier Type .
```

identifier为新类型的名称（type name），与Type有相同的underlying type。type上定义的操作都可以应用于新类型上，新的类型与type是不相同的。

那么对于类型声明：`type IntArray [16]int`，其TypeSpec就是`IntArray [16]int`。

**underlying type**

文档中的说明：[Types](https://golang.org/ref/spec#Types)

每个类型T都有一个underlying type：如果类型T是预定义的类型（boolean、numeric、string types、type literal），那么underlying type就是T；否则，T的underlying type就是T在其类型声明（TypeDecl）中所指向的类型的underlying type。

例如：

```
   type T1 string
   type T2 T1
   type T3 []T1
   type T4 T3
```

string、T1、T2的underlying type是string，[]T1，T3，T4的underlying type是[]T1。

**type literal**

文档中的说明：[TypeLit](https://golang.org/ref/spec#TypeLit)

```
Type      = TypeName | TypeLit | "(" Type ")" .
TypeName  = identifier | QualifiedIdent .
TypeLit   = ArrayType | StructType | PointerType | FunctionType | InterfaceType |
        SliceType | MapType | ChannelType .
```

TypeLit是type literal的缩写，用于表示复合类型 。

**named types、unnamed types**

文档中的说明：[Types](https://golang.org/ref/spec#Types)

named types由type name（包括被包名修饰的）指定。

unnamed types由类型字面量（type literal）指定。

例如：`int`是named types，`[]int`是unmanned types。

赋值例子：

```
type AINT int
type BINT int

var a AINT = 0
var b BINT = 1
var a1 AINT = 2

a = b //非法赋值，a、b的类型都是named type，但来自不同的TypeSpec
a = a1 //合法赋值，a、a1的类型都是named type，来自相同的TypeSpec

type ARRINT [10]int
type ANOTHERINT int

var a ARRINT
var b [10]int
var c [10]int
var d [10]ANOTHERINT

a = b //a的类型是named type，b的类型是unnamed type，对于规则1来说是不合法的赋值，但却满足规则2，所以还是合法的赋值
b = c //合法赋值，b、c的类型都是unnamed type，并且它们的TypeLit是相同的
c = d //非法赋值，c、d的类型都是unnamed type，但它们的TypeLit是不同的
```

2、x的类型V和T有相同的underlying type，并且至少V、T中有一个不是named type

```
type ARRINT [7]int

var a ARRINT
var b [7]int

a = b //合法赋值，a为named type，b为unnamed type，并且它们的underlying type相同
```

3、T是interface类型，并且x实现了T

```
type Person interface {
    say()
}

type Student struct {
    name string
}

func (s Student) say() {
    fmt.Println("Hello, I am " + s.name)
}

var bob Student = Student{name: "bob"}
var p Person
p = bob //合法赋值
```

4、x是双向管道类型的值，T是管道类型，x的类型V和T是相同的element type，并且至少V、T中有一个不是named type

```
var c1, c2 chan int
type CHANINT chan int
type CHANSTRING chan string
var c3 CHANINT
c1 = c2 //合法赋值，类型是相同的
c1 = c3 //合法赋值，c1是unnamed type，c3是named type，并且element type相同
```

```
var c1 chan int
var c2 chan <- int
c2 = c1 //合法赋值，c1是双向管道类型，c2是单向只写管道类型，并且element type都为int
c1 = c2 //非法赋值
```

**ElementType**

文档中的说明：[ElementType](https://golang.org/ref/spec#ElementType)

```
//以下类型的element type示例
[10]int //int
[]string //string
map[string]int //int
chan int //int
```

5、x是预定义的标识符nil，T是pointer、function、slice、map、channel、interface类型

6、x是常量，并且能代表类型T的某个值

```
const STRING = "Hello"
var s string = STRING //合法赋值
var i int = STRING //非法赋值
```