---
layout: post
title: "golang singleflight 用武之地"
description: "golang singleflight 用武之地"
category:
tags: []
---
{% include JB/setup %}

#### 缓存更新问题
- - -
当缓存失效时，需要去数据存储层获取数据，然后存储到缓存中。

通常缓存更新方案：

1. 业务代码中，根据key从缓存拿不到数据，访问存储层获取数据后更新缓存
1. 由专门的定时脚本在缓存失效前对其进行更新
1. 通过分布式锁，实现只有一个请求负责缓存更新，其他请求等待：[一种基于哨兵的缓存访问策略](http://yangxikun.github.io/%E7%BC%93%E5%AD%98/2015/07/02/cache-access.html)

#### 服务中某个接口请求量暴增问题
- - -
比如某个帖子突然很火，帖子下有非常多的跟帖回复，负责提供帖子内容、回帖内容的接口，对于该帖子的请求量就会非常多。

如果每个请求都落到下游服务，通常会导致下游服务瞬时负载升高。如果使用缓存，如何判断当前接口请求的内容需要缓存下来？缓存的过期、更新问题？

<!--more-->

#### golang singleflight
- - -
该库提供了一个简单有效的方案应对上面提到的问题，初次见识到 singleflight 是在 [golang/groupcache](https://github.com/golang/groupcache) 中。

groupcache 缓存更新能够做到对同一个失效key的多个请求，只有一个请求执行对key的更新操作，其文档相关描述如下：

> comes with a cache filling mechanism. Whereas memcached just says "Sorry, cache miss", often resulting in a thundering herd of database (or whatever) loads from an unbounded number of clients (which has resulted in several fun outages), groupcache coordinates cache fills such that only one load in one process of an entire replicated set of processes populates the cache, then multiplexes the loaded value to all callers.

从 singleflight 的 test 可以了解到其用法：

```go
func TestDoDupSuppress(t *testing.T) {
	var g Group
	c := make(chan string)
	var calls int32
	fn := func() (interface{}, error) {
		atomic.AddInt32(&calls, 1)
		return <-c, nil
	}

	const n = 10
	var wg sync.WaitGroup
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() { // n个协程同时调用了g.Do，fn中的逻辑只会被一个协程执行
			v, err := g.Do("key", fn)
			if err != nil {
				t.Errorf("Do error: %v", err)
			}
			if v.(string) != "bar" {
				t.Errorf("got %q; want %q", v, "bar")
			}
			wg.Done()
		}()
	}
	time.Sleep(100 * time.Millisecond) // let goroutines above block
	c <- "bar"
	wg.Wait()
	if got := atomic.LoadInt32(&calls); got != 1 {
		t.Errorf("number of calls = %d; want 1", got)
	}
}
```

该测试用例中，只有1个协程执行了fn，其他9个协程能拿到fn执行后的返回结果。即fn只执行了1次，但其结果会返回给多个协程。

看下 singleflight 是如何做到这一点的：

```go
// call is an in-flight or completed Do call
type call struct {
	wg  sync.WaitGroup
	val interface{}
	err error
}
```

call 用来表示一个正在执行或已完成的函数调用。

```go
// Group represents a class of work and forms a namespace in which
// units of work can be executed with duplicate suppression.
type Group struct {
	mu sync.Mutex       // protects m
	m  map[string]*call // lazily initialized
}
```

Group 可以看做是任务的分类。

```go
// Do executes and returns the results of the given function, making
// sure that only one execution is in-flight for a given key at a
// time. If a duplicate comes in, the duplicate caller waits for the
// original to complete and receives the same results.
func (g *Group) Do(key string, fn func() (interface{}, error)) (interface{}, error) {
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}
	if c, ok := g.m[key]; ok {
		g.mu.Unlock()
		c.wg.Wait()
		return c.val, c.err
	}
	c := new(call)
	c.wg.Add(1)
	g.m[key] = c
	g.mu.Unlock()

	c.val, c.err = fn()
	c.wg.Done()

	g.mu.Lock()
	delete(g.m, key)
	g.mu.Unlock()

	return c.val, c.err
}
```

在 Do 函数的源码中，g.m 的读写被 g.mu 互斥锁保护，fn 的返回结果存储在 call.val、call.err 中，通过 sync.WaitGroup 实现等待 fn 执行结束。

回到本文开头提到的问题，对于缓存的更新，可以这样实现：

```go
if (cacheMiss) {
    fn = func() (interface{}, error) {
        // 缓存更新逻辑
    }
    data, err = g.Do(cacheKey, fn)
}
```

对于防止暴增的接口请求对下游服务造成瞬时高负载，可以这样实现：

```go
fn = func() (interface{}, error) {
    // 发送请求到其他服务接口
}
data, err = g.Do(apiNameWithParams, fn)
```
