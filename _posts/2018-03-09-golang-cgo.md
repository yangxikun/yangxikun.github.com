---
layout: post
title: "golang cgo 开发小结"
description: ""
category: GoLang
tags: []
---
{% include JB/setup %}

工作上遇到一个需求，需要把一个C++的动态库的功能封装为Web接口。由于没有C++开发经验，C有点经验，于是考虑了两种方案：

1. 封装为PHP扩展
1. 在Golang中使用CGO

两种方案我都可以做，但最终决定采用第2种方案，主要考虑的因素是这个Web服务最终需要在客户那里进行私有化部署，采用PHP的话，部署的时候还需要Nginx、Fpm（当然也可以直接用Swoole），但是PHP代码是明文的，虽然可以买一些商业软件进行加密（比如Swoole Compiler）。如果直接用Golang的话，就可以直接给用户部署一个二进制程序（需要strip掉符号信息）就可以了，部署起来更方便。

下面将通过一个示例程序，演示如何在Golang中通过cgo调用C++。

<!--more-->

示例代码目录：

```plaintext
.
├── bin
│   └── cgo
└── src
    └── cgo
        ├── c_src.cpp   // 在Golang中调用的C函数定义
        ├── c_src.h     // C头文件，声明了哪些C函数会在Golang中使用，在main.go中包含
        ├── main.go
        ├── src.cpp     // C++代码
        └── src.hpp     // C++头文件
```

c_src.h 源码：

```c
#ifndef WRAP_CPP_H
#define WRAP_CPP_H

#ifdef __cplusplus
extern "C" {
#endif // __cplusplus

typedef void* Foo;
Foo FooNew();
void FooDestroy(Foo f);
const char* FooGetName(Foo f, int* retLen);
void FooSetName(Foo f, char* name);

#ifdef __cplusplus
}
#endif // __cplusplus

#endif // WRAP_CPP_H
```

`extern "C"`作用：[Combining C++ and C - how does #ifdef __cplusplus work?
](https://stackoverflow.com/questions/3789340/combining-c-and-c-how-does-ifdef-cplusplus-work)

c_src.cpp 源码：

```cpp
#include "src.hpp"
#include "c_src.h"
#include <cstring>

// 返回cxxFoo对象，但转换为void*
Foo FooNew()
{
    cxxFoo* ret = new cxxFoo("rokety");
    return (void*)ret;
}

void FooDestroy(Foo f)
{
    cxxFoo* foo = (cxxFoo*)f;
    delete foo;
}

// 封装cxxFoo的get_name方法
const char* FooGetName(Foo f, int* ret_len)
{
    cxxFoo* foo = (cxxFoo*)f;
    std::string name = foo->get_name();
    *ret_len = name.length();
    const char* ret_str = (const char*)malloc(*ret_len);
    memcpy((void*)ret_str, name.c_str(), *ret_len);
    return ret_str;
}

// 封装cxxFoo的set_name方法
void FooSetName(Foo f, char* name)
{
    cxxFoo* foo = (cxxFoo*)f;
    std::string _name(name, strlen(name));
    foo->set_name(_name);
}
```

c_src.cpp 可能的疑问：

* 为何需要定义Foo？因为在C中没有Class的概念，所以需要把C++的Class转换为C中的数据类型
* 为何在FooGetName中需要进行malloc和memcpy？因为name是局部变量，并且内存分配在栈上，当cgo调用返回后，name所占用的内存会被释放掉。

main.go 源码：

```go
package main

// #include "c_src.h"
// #include <stdlib.h>
import "C"

import (
	"fmt"
	"unsafe"
)

type GoFoo struct {
	foo C.Foo
}

func NewGoFoo() GoFoo {
	var ret GoFoo
	ret.foo = C.FooNew()
	return ret
}

func (f GoFoo) Destroy() {
	C.FooDestroy(f.foo)
}

func (f GoFoo) GetName() string {
	rLen := C.int(0)
	name := C.FooGetName(f.foo, &rLen)
	defer C.free(unsafe.Pointer(name))  // 必须使用C的free函数，释放FooGetName中malloc的内存
	return C.GoStringN(name, rLen)      // 从name构造出golang的string类型值
}

func (f GoFoo) SetName(name string) {
	cname := C.CString(name)        // 将golang的string类型值转换为c中的char*类型值，这里会调用到c的malloc
	C.FooSetName(f.foo, cname)
	C.free(unsafe.Pointer(cname))   // 释放上面malloc的内存
}

func main() {
	foo := NewGoFoo()
	fmt.Println(foo.GetName())
	foo.GetName()
	foo.SetName("new rokety")
	fmt.Println(foo.GetName())
	foo.Destroy()
}
```

main.go 可能的疑问：

* unsafe.Pointer(...)相当于把变量强转为C中的void*类型
* SetName中为何需要做转换，因为name变量的内存是在Golang中分配的，且string类型是不可修改的，因此，需要在c中分配name所需要的内存，以便在FooSetName中使用
* 需要注意的一点是`import "C"`上面必须紧跟`// #include ...`注释

src.hpp 源码：

```cpp
#ifndef CXX_H
#define CXX_H

#include <string>

class cxxFoo
{
public:
    cxxFoo(std::string name);
    ~cxxFoo();
    std::string get_name();
    void set_name(std::string name);

private:
    std::string name;
};

#endif // CXX_H
```

src.cpp 源码

```cpp
#include "src.hpp"
#include <iostream>

cxxFoo::cxxFoo(std::string name)
{
    this->name = name;
}

cxxFoo::~cxxFoo()
{
}

std::string cxxFoo::get_name()
{
    return this->name;
}

void cxxFoo::set_name(std::string name)
{
    this->name = name;
}
```

小结：

* C中的数据类型会与Golang的C.xxx数据类型对应：[CGO 类型（CGO Types）](http://www.cnblogs.com/majianguo/p/7650059.html)
* 在C/C++中申请的内存，就得在C/C++中释放
* 对于需要链接C/C++动态库，或加上编译参数，可以在`import "C"`加上对应注释`// #cgo CFLAGS: -DPNG_DEBUG=1`

参考资料：

* [How to use C++ in Go?](https://stackoverflow.com/questions/1713214/how-to-use-c-in-go)
* [Command cgo](https://golang.org/cmd/cgo/)
* [C? Go? Cgo!](https://blog.golang.org/c-go-cgo)
* [Golang CGO编程之调用返回char*指针及长度的C函数库](http://blog.codeg.cn/post/blog/2016-04-20-golang-cgo/)
* [CGO: Go与C互操作技术（一）：Go调C基本原理](https://bbs.huaweicloud.com/blogs/2c0a55d8c85811e7b8317ca23e93a891)