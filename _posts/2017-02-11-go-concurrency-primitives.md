---
layout: post
title: "golang并发编程的理解"
description: "golang并发编程的理解"
category: GoLang
tags: [GoLang并发]
---
{% include JB/setup %}


golang通过语言层面的特性goroutine和channel提供了优雅的并发编程方式，[Share Memory By Communicating](https://blog.golang.org/share-memory-by-communicating)一文中有这么句“名言”：

`Do not communicate by sharing memory; instead, share memory by communicating.`

单纯看这句话其实不好理解，但结合文中的例子算是基本明白其中的含义了。

文中以编写并发拉取给定的URL资源程序为例子，展示了传统多线程并发编程模型和golang并发编程模型的不同。

<!--more-->
传统多线程并发编程模型下，拉取程序代码大致如下：

```go
type Resource struct {
    url        string // 需要拉取的资源的URL
    polling    bool   // 是否正在拉取
    lastPolled int64  // 最新的拉取完成时间
}

type Resources struct {
    data []*Resource // 待拉取资源列表
    lock *sync.Mutex // 同步锁
}

func Poller(res *Resources) {
    for {
        // get the least recently-polled Resource
        // and mark it as being polled
        res.lock.Lock()
        var r *Resource
        for _, v := range res.data {
            if v.polling {
                continue
            }
            if r == nil || v.lastPolled < r.lastPolled {
                r = v
            }
        }
        if r != nil {
            r.polling = true
        }
        res.lock.Unlock()
        if r == nil {
            continue
        }

        // poll the URL

        // update the Resource's polling and lastPolled
        res.lock.Lock()
        r.polling = false
        r.lastPolled = time.Nanoseconds()
        res.lock.Unlock()
    }
}
```

如果使用golang实现，拉取程序代码大致如下：

```go
type Resource string

func Poller(in, out chan *Resource) {
    for r := range in {
        // poll the URL

        // send the processed Resource to out
        out <- r
    }
}
```

结合这两段代码，我对文中那句“名言”的理解：

传统多线程并发编程模型下，Poller接收的参数是需要被多个线程共享的资源（调用类似：pthread_create(thread_id, attr, Poller, resources)），而多个线程之间的同步依赖资源上的锁，这就是通过共享资源进行同步（communicate by sharing memory）；而golang并发编程模型下，Poller接收的参数是两个channel（调用类似：go Poller(in, out)），只管从in channel中获取需要拉取的资源的URL，拉取完后，放入out channel，这两个channel就是整个并发程序中的通信载体，goroutine通过channel共享待拉取的资源的URL，这就是通过同步进行资源共享（share memory by communicating）。

其实在传统多线程并发编程模型下也可以实现golang的这个编程模型，例如封装一个channel出来：

```go
type Resource string

struct In {
    data []Resource
    lock *sync.Mutex
    close bool
}

func (*in)Pop() Resource {
	res = ""
Loop:
	in.lock.Lock()
	if len(in.data) > 0 {
		res = in.data[0]
		in.data = in.data[1:]
	} else if(!in.close) {
		in.lock.Unlock()
		sleep
		goto Loop
	}
	in.lock.Unlock()
	return res
}

func (*in)Push(res Resource) {
	in.lock.Lock()
	in.data = append(in.data, res)
	in.lock.Unlock()
}

func (*in)Close() {
	in.close = true
}

// Out 类似

func Poller(in In, out Out) {
    for r := in.Pop(); r != ""; r = in.Pop() {
        // poll the URL

        // send the processed Resource to out
        out.Push(r)
    }
}

// 调用类似：pthread_create(thread_id, attr, Poller, in, out)
```
golang其实是在语言层面上通过goroutine和channel隐藏了这些东西。
