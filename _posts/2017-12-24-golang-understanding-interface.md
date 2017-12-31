---
layout: post
title: "Golang understanding interface"
description: "Golang understanding interface"
category: GoLang
tags: []
---
{% include JB/setup %}

本文简单总结自[Understanding Go Interfaces](https://www.youtube.com/watch?v=F4wUrj6pmSI)，对应的[PPT](https://speakerdeck.com/campoy/understanding-the-interface)。

> In object-oriented programming, a protocol or interface is a common means for unrelated objects to communicate with each other.

<!--more-->

#### concrete types
* 描述了内存布局

![](/assets/img/201712240101.png)

* 通过方法将行为绑定到数据上

```go
type Number int
func (n Number) Positive() bool {
    return n > 0
}
```

#### abstract types
* 描述了行为

![](/assets/img/201712240102.png)

* 定义了方法集

```go
type Positiver interface {
    Positive() bool
}
```

#### 为何使用interface？
1、编写通用的算法

```go
// A type, typically a collection, that satisfies sort.Interface can be
// sorted by the routines in this package. The methods require that the
// elements of the collection be enumerated by an integer index.
type Interface interface {
	// Len is the number of elements in the collection.
	Len() int
	// Less reports whether the element with
	// index i should sort before the element with index j.
	Less(i, j int) bool
	// Swap swaps the elements with indexes i and j.
	Swap(i, j int)
}
```

2、隐藏实现细节

![](/assets/img/201712240103.png)

3、提供拦截点

![](/assets/img/201712240104.png)

```go
type headers struct {
    rt http.RoundTripper
    v map[string]string
}

func (h headers) RoundTrip(r *http.Request) (*http.Response, error) {
    for key, value := range h.v {
        r.Header.Set(key, value)
    }
    return h.rt.RoundTrip(r)
}

func main() {
    c := &http.Client{
        Transport: headers{
            rt: http.DefaultTransport,
            v: map[string]string{"foo": "bar"},
        },
    }

    res, err := c.Get("http://golang.org")
}
```

> The bigger the interface, the weaker the abstraction. - Rob Pike

方法集越大，抽象程度越低。事物的抽象，是从众多的事物中抽取出共同的、本质性的特征。抽象程度越高，涵盖的事物也就越多，相应的方法集也会越少。

> Be conservative in what you do/send, be liberal in what you accept from others. - Robustness Principle

在方法定义和实现的时候应当遵行的原则：对于内部所做的/返回的尽量保守，对于从外部接收的尽量开放。

> Return concrete types, receive interface as parameters.

Golang interface与其他语言的interface不同点主要在于Golang interface是隐式实现的，而其他语言需要显示的implement。