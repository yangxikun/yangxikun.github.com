---
layout: post
title: "golang grpc 客户端负载均衡、重试、健康检查"
description: "golang grpc 客户端负载均衡、重试、健康检查"
category: GoLang
tags: []
---
{% include JB/setup %}

#### Go GRPC 客户端是如何管理与服务端的连接的？

grpc.ClientConn 表示一个客户端实例与服务端之间的连接，主要包含如下数据结构：

##### grpc.connectivityStateManager(grpc.ClientConn.csMgr) 总体的连接状态

状态类型为 connectivity.State，有如下几种状态：
* Idle
* Connecting
* Ready
* TransientFailure
* Shutdown

grpc.ClientConn 包含了多个 grpc.addrConn（每个 grpc.addrConn 表示客户端到一个服务端的一条连接），每个 grpc.addrConn 也有自己的连接转态。
* 当至少有一个 grpc.addrConn.state = Ready，则 grpc.ClientConn.csMgr.state = Ready
* 当至少有一个 grpc.addrConn.state = Connecting，则 grpc.ClientConn.csMgr.state = Connecting
* 否则 grpc.ClientConn.csMgr.state = TransientFailure

> 默认实现下客户端与某一个服务端（host:port）只会建立一条连接，所有 RPC 执行都会复用这条连接。
> 关于为何只建立一条连接可以看下这个 issue：[Use multiple connections to avoid the server's SETTINGS_MAX_CONCURRENT_STREAMS limit #11704](https://github.com/grpc/grpc/issues/11704)
> 不过如果使用 manual.Resolver，把同一个服务地址复制多遍，也能做到与一个服务端建立多个连接。

关于 grpc.addrConn.state 的转态切换可参考设计文档：[gRPC Connectivity Semantics and API](https://github.com/grpc/grpc/blob/master/doc/connectivity-semantics-and-api.md)

<!--more-->

##### grpc.ccResolverWrapper 服务端地址解析模块的封装

grpc 内置的 resolver.Resolver 有：
* dns.dnsResolver：通过域名解析服务地址
* manual.Resolver：手动设置服务地址
* passthrough.passthroughResolver：将 grpc.Dial 参数中的地址作为服务地址，这也是默认的

##### grpc.ccBalancerWrapper 负载均衡模块的封装

grpc 内置的 balancer.Balancer 有：
* grpc.pickfirstBalancer：只使用一个服务地址
* roundrobin：在多个服务地址中轮转
* grpclb：使用一个单独的服务提供负载均衡信息（可用的服务地址列表）

可参考设计文档：[Load Balancing in gRPC](https://github.com/grpc/grpc/blob/master/doc/load-balancing.md)

##### grpc.pickerWrapper 从连接池中选择一个连接

与使用的 balancer.Balancer 具体实现相关：
* grpc.pickfirstBalancer：grpc.picker，返回当前的连接
* roundrobin：roundrobin.rrPicker，轮转返回一个可用连接
* grpclb：grpclb.lbPicker，轮转返回一个可用连接

##### 配置 resolver = dns, balancer = roundrobin 组件之间的关系如下图：

![](/assets/img/201910190102.jpeg)

* dns.dnsResolver 会启动一个 goroutine，负责解析服务地址，发送更新事件
* grpc.ccBalancer.Wrapper 会启动一个 goroutine，负责监听服务地址更新事件和连接状态变化事件
    * 服务地址更新事件会触发负载均衡组件更新连接池
    * 连接状态变化事件会触发负载均衡组件更新连接池中连接的状态，以及更新picker
* 当执行 RPC 调用时，会通过 grpc.ClientConn.blockingpicker（即 grpc.pickerWrapper.pick，最终调用 roundrobin.rrPicker.Pick）从连接池中获取连接

#### 代码实现

grpc 内置的 dns resolver 定时进行 dns 解析的时间是1830秒，这个时间对于实际应用来说太长了，对服务端进行扩容需要等待半个小时才生效。然而官方实现并没有暴露接口来设置这个超时时间，所以我这里把 `resolver/dns/dns_resolver.go` 代码复制了一份，修改了下跟定时相关的常量 `defaultFreq` 和变量 `minDNSResRate`。

> 笔者使用的 grpc golang 客户端版本是 google.golang.org/grpc v1.24.0

客户端代码：

```go
package main

import (
    "context"
    "flag"
    "fmt"
    "log"
    "time"

    pb "github.com/yangxikun/go-grpc-client-side-lb-example/pb"
    _ "github.com/yangxikun/go-grpc-client-side-lb-example/resolver/dns"
    "google.golang.org/grpc"
    "google.golang.org/grpc/balancer/roundrobin"
    "google.golang.org/grpc/resolver"
)

const (
    defaultName = "rokety"
)

func main() {
    log.SetFlags(log.Lshortfile | log.Ldate)
    var address string
    var timeout int
    flag.IntVar(&timeout, "timeout", 1, "greet rpc call timeout")
    flag.StringVar(&address, "address", "localhost:50051", "grpc server addr")
    flag.Parse()
    // Set resolver
    resolver.SetDefaultScheme("custom_dns")
    // Set up a connection to the server.
    conn, err := grpc.Dial(address, grpc.WithInsecure(),
        grpc.WithDefaultServiceConfig(fmt.Sprintf(`{"LoadBalancingPolicy": "%s"}`, roundrobin.Name)),
        grpc.WithBlock(), grpc.WithBackoffMaxDelay(time.Second))
    if err != nil {
        log.Fatalf("did not connect: %v", err)
    }
    defer conn.Close()
    c := pb.NewGreeterClient(conn)

    // Contact the server and print out its response.
    for range time.Tick(time.Second) {
        ctx, cancel := context.WithTimeout(context.Background(), time.Duration(timeout)*time.Second)
        r, err := c.SayHello(ctx, &pb.HelloRequest{Name: defaultName})
        if err != nil {
            log.Printf("could not greet: %v\n", err)
        } else {
            log.Printf("Greeting: %s", r.Message)
        }
        cancel()
    }
}
```

服务端代码：

> 服务端启动时会随机设置接口返回延迟时间为0或1秒

```go
package main

import (
	"context"
	"log"
	"math/rand"
	"net"
	"time"

	pb "github.com/yangxikun/go-grpc-client-side-lb-example/pb"
	"google.golang.org/grpc"
	"google.golang.org/grpc/reflection"
)

const (
	port = ":50051"
)

var stuckDuration time.Duration

// server is used to implement helloworld.GreeterServer.
type server struct{}

// SayHello implements helloworld.GreeterServer
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	time.Sleep(stuckDuration)
	return &pb.HelloReply{Message: "Hello " + in.Name + "! From " + GetIP()}, nil
}

func main() {
	// simulate busy server
	stuckDuration = time.Duration(rand.NewSource(time.Now().UnixNano()).Int63()%2) * time.Second

	if stuckDuration == time.Second {
		log.Println("I will stuck one second!!!")
	}

	lis, err := net.Listen("tcp", port)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	s := grpc.NewServer()
	pb.RegisterGreeterServer(s, &server{})
	// Register reflection service on gRPC server.
	reflection.Register(s)
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}

func GetIP() string {
	ifaces, _ := net.Interfaces()
	// handle err
	for _, i := range ifaces {
		addrs, _ := i.Addrs()
		// handle err
		for _, addr := range addrs {
			var ip net.IP
			switch v := addr.(type) {
			case *net.IPNet:
				ip = v.IP
			case *net.IPAddr:
				ip = v.IP
			default:
				continue
			}
			if ip.String() != "127.0.0.1" {
				return ip.String()
			}
		}
	}
	return ""
}
```

接下来将借助 k8s 部署客户端和服务端代码，测试代码实际效果。

启动3个服务端实例，客户端 timeout 设置为2秒，客户端日志情况：

```text
2019/10/21 main.go:46: Greeting: Hello rokety! From 10.200.235.181
2019/10/21 main.go:46: Greeting: Hello rokety! From 10.200.251.161
2019/10/21 main.go:46: Greeting: Hello rokety! From 10.200.105.163
2019/10/21 main.go:46: Greeting: Hello rokety! From 10.200.235.181
2019/10/21 main.go:46: Greeting: Hello rokety! From 10.200.251.161
2019/10/21 main.go:46: Greeting: Hello rokety! From 10.200.105.163
2019/10/21 main.go:46: Greeting: Hello rokety! From 10.200.235.181
2019/10/21 main.go:46: Greeting: Hello rokety! From 10.200.251.161
2019/10/21 main.go:46: Greeting: Hello rokety! From 10.200.105.163
```

将服务端缩容为1个实例后，客户端日志情况：

```go
2019/10/21 main.go:46: Greeting: Hello rokety! From 10.200.235.181
2019/10/21 main.go:46: Greeting: Hello rokety! From 10.200.251.161
2019/10/21 main.go:46: Greeting: Hello rokety! From 10.200.105.163
2019/10/21 main.go:44: could not greet: rpc error: code = Unavailable desc = transport is closing
2019/10/21 main.go:46: Greeting: Hello rokety! From 10.200.105.163
2019/10/21 main.go:46: Greeting: Hello rokety! From 10.200.105.163
2019/10/21 main.go:46: Greeting: Hello rokety! From 10.200.105.163
2019/10/21 main.go:46: Greeting: Hello rokety! From 10.200.105.163
2019/10/21 main.go:46: Greeting: Hello rokety! From 10.200.105.163
2019/10/21 main.go:46: Greeting: Hello rokety! From 10.200.105.163
2019/10/21 main.go:46: Greeting: Hello rokety! From 10.200.105.163
```

可以看到客户端遇到了一次连接被关闭的错误，为了避免缩容和滚动更新导致的此类错误，我们可以在客户端的代码里加上重试机制：

grpc 客户端的重试策略有2种实现，具体可参考涉及文档：[gRPC Retry Design](https://github.com/grpc/proposal/blob/master/A6-client-retries.md)：

* Retry policy：出错时立即重试
* Hedging policy：定时发送并发的多个请求，根据请求的响应情况决定是否发送下一个同样的请求，还是返回（该策略目前未实现）

> 注意：
>
> 客户端程序启动时，还需要设置环境变量：GRPC_GO_RETRY=on
>
> MaxAttempts = 2，即最多尝试2次，也就是最多重试1次
>
> RetryableStatusCodes 只设置了 UNAVAILABLE，也就是解决上面出现的错误：`rpc error: code = Unavailable desc = transport is closing`
>
> RetryableStatusCodes 中设置 DeadlineExceeded 和 Canceled 是没有作用的，因为在重试逻辑的代码里判断到 Context 超时或取消就会立即退出重试逻辑了。

添加重试逻辑后的客户端代码：

```go
package main

import (
	"context"
	"flag"
	"fmt"
	"log"
	"time"

	pb "github.com/yangxikun/go-grpc-client-side-lb-example/pb"
	_ "github.com/yangxikun/go-grpc-client-side-lb-example/resolver/dns"
	"google.golang.org/grpc"
	"google.golang.org/grpc/balancer/roundrobin"
	"google.golang.org/grpc/resolver"
)

const (
	defaultName = "rokety"
)

func main() {
	log.SetFlags(log.Lshortfile | log.Ldate | log.Ltime)
	var address string
	var timeout int
	flag.IntVar(&timeout, "timeout", 1, "greet rpc call timeout")
	flag.StringVar(&address, "address", "localhost:50051", "grpc server addr")
	flag.Parse()
	// Set up a connection to the server.
	resolver.SetDefaultScheme("custom_dns")
	conn, err := grpc.Dial(address, grpc.WithInsecure(),
		grpc.WithDefaultServiceConfig(fmt.Sprintf(`{"LoadBalancingPolicy": "%s","MethodConfig": [{"Name": [{"Service": "helloworld.Greeter"}], "RetryPolicy": {"MaxAttempts":2, "InitialBackoff": "0.1s", "MaxBackoff": "1s", "BackoffMultiplier": 2.0, "RetryableStatusCodes": ["UNAVAILABLE"]}}]}`, roundrobin.Name)),
		grpc.WithBlock(), grpc.WithBackoffMaxDelay(time.Second))
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	c := pb.NewGreeterClient(conn)

	// Contact the server and print out its response.
	for range time.Tick(time.Second) {
		ctx, cancel := context.WithTimeout(context.Background(), time.Duration(timeout)*time.Second)
		r, err := c.SayHello(ctx, &pb.HelloRequest{Name: defaultName})
		if err != nil {
			log.Printf("could not greet: %v\n", err)
		} else {
			log.Printf("Greeting: %s", r.Message)
		}
		cancel()
	}
}
```

服务端实例恢复为3个，修改客户端 timeout 设置为1秒后，客户端日志情况：

```text
2019/10/21 13:02:54 main.go:46: Greeting: Hello rokety! From 10.200.38.220
2019/10/21 13:02:56 main.go:44: could not greet: rpc error: code = DeadlineExceeded desc = context deadline exceeded
2019/10/21 13:02:57 main.go:44: could not greet: rpc error: code = DeadlineExceeded desc = context deadline exceeded
2019/10/21 13:02:57 main.go:46: Greeting: Hello rokety! From 10.200.38.220
2019/10/21 13:02:59 main.go:44: could not greet: rpc error: code = DeadlineExceeded desc = context deadline exceeded
2019/10/21 13:03:00 main.go:44: could not greet: rpc error: code = DeadlineExceeded desc = context deadline exceeded
```

可以看到有2个服务端的响应是一直超时的，但实际业务使用中，希望避免这种错误，这时可以使用 grpc 的健康检查功能（设计文档：[GRPC Health Checking Protocol](https://github.com/grpc/grpc/blob/master/doc/health-checking.md)），该功能要求服务端实现健康检查接口，客户端和服务端的代码都需要调整：

客户端代码：

> 注意：
>
> 需要在 grpc.WithDefaultServiceConfig 中配置 HealthCheckConfig
>
> 需要导入`_ "google.golang.org/grpc/health"`

```go
package main

import (
	"context"
	"flag"
	"fmt"
	"log"
	"time"

	pb "github.com/yangxikun/go-grpc-client-side-lb-example/pb"
	_ "github.com/yangxikun/go-grpc-client-side-lb-example/resolver/dns"
	"google.golang.org/grpc"
	"google.golang.org/grpc/balancer/roundrobin"
	_ "google.golang.org/grpc/health"
	"google.golang.org/grpc/resolver"
)

const (
	defaultName = "rokety"
)

func main() {
	log.SetFlags(log.Lshortfile | log.Ldate | log.Ltime)
	var address string
	var timeout int
	flag.IntVar(&timeout, "timeout", 1, "greet rpc call timeout")
	flag.StringVar(&address, "address", "localhost:50051", "grpc server addr")
	flag.Parse()
	// Set up a connection to the server.
	resolver.SetDefaultScheme("custom_dns")
	conn, err := grpc.Dial(address, grpc.WithInsecure(),
		grpc.WithDefaultServiceConfig(fmt.Sprintf(`{"LoadBalancingPolicy": "%s","MethodConfig": [{"Name": [{"Service": "helloworld.Greeter"}], "RetryPolicy": {"MaxAttempts":2, "InitialBackoff": "0.1s", "MaxBackoff": "1s", "BackoffMultiplier": 2.0, "RetryableStatusCodes": ["UNAVAILABLE", "CANCELLED"]}}], "HealthCheckConfig": {"ServiceName": "helloworld.Greeter"}}`, roundrobin.Name)),
		grpc.WithBlock(), grpc.WithBackoffMaxDelay(time.Second))
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	c := pb.NewGreeterClient(conn)

	// Contact the server and print out its response.
	for range time.Tick(time.Second) {
		ctx, cancel := context.WithTimeout(context.Background(), time.Duration(timeout)*time.Second)
		r, err := c.SayHello(ctx, &pb.HelloRequest{Name: defaultName})
		if err != nil {
			log.Printf("could not greet: %v\n", err)
		} else {
			log.Printf("Greeting: %s", r.Message)
		}
		cancel()
	}
}
```

服务端代码：

> 注意：
>
> 实现 grpc_health_v1 的接口
>
> 注册到服务中：`grpc_health_v1.RegisterHealthServer(s, &healthServer{})`

```go
package main

import (
	"context"
	"log"
	"math/rand"
	"net"
	"time"

	pb "github.com/yangxikun/go-grpc-client-side-lb-example/pb"
	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/health/grpc_health_v1"
	"google.golang.org/grpc/reflection"
	"google.golang.org/grpc/status"
)

const (
	port = ":50051"
)

var stuckDuration time.Duration

type healthServer struct{}

func (h *healthServer) Check(ctx context.Context, req *grpc_health_v1.HealthCheckRequest) (*grpc_health_v1.HealthCheckResponse, error) {
	log.Println("recv health check for service:", req.Service)
	if stuckDuration == time.Second {
		return &grpc_health_v1.HealthCheckResponse{Status: grpc_health_v1.HealthCheckResponse_NOT_SERVING}, nil
	}
	return &grpc_health_v1.HealthCheckResponse{Status: grpc_health_v1.HealthCheckResponse_SERVING}, nil
}

func (h *healthServer) Watch(req *grpc_health_v1.HealthCheckRequest, stream grpc_health_v1.Health_WatchServer) error {
	log.Println("recv health watch for service:", req.Service)
	resp := new(grpc_health_v1.HealthCheckResponse)
	if stuckDuration == time.Second {
		resp.Status = grpc_health_v1.HealthCheckResponse_NOT_SERVING
	} else {
		resp.Status = grpc_health_v1.HealthCheckResponse_SERVING
	}
	for range time.NewTicker(time.Second).C {
		err := stream.Send(resp)
		if err != nil {
			return status.Error(codes.Canceled, "Stream has ended.")
		}
	}
	return nil
}

// server is used to implement helloworld.GreeterServer.
type server struct{}

// SayHello implements helloworld.GreeterServer
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	time.Sleep(stuckDuration)
	return &pb.HelloReply{Message: "Hello " + in.Name + "! From " + GetIP()}, nil
}

func main() {
	// simulate busy server
	stuckDuration = time.Duration(rand.NewSource(time.Now().UnixNano()).Int63()%2) * time.Second

	if stuckDuration == time.Second {
		log.Println("I will stuck one second!!!")
	}

	lis, err := net.Listen("tcp", port)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	s := grpc.NewServer()
	pb.RegisterGreeterServer(s, &server{})
	grpc_health_v1.RegisterHealthServer(s, &healthServer{})
	// Register reflection service on gRPC server.
	reflection.Register(s)
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}

func GetIP() string {
	ifaces, _ := net.Interfaces()
	// handle err
	for _, i := range ifaces {
		addrs, _ := i.Addrs()
		// handle err
		for _, addr := range addrs {
			var ip net.IP
			switch v := addr.(type) {
			case *net.IPNet:
				ip = v.IP
			case *net.IPAddr:
				ip = v.IP
			default:
				continue
			}
			if ip.String() != "127.0.0.1" {
				return ip.String()
			}
		}
	}
	return ""
}
```

更新服务镜像，查看客户端日志：

```text
2019/10/21 14:25:01 main.go:47: Greeting: Hello rokety! From 10.200.251.170
2019/10/21 14:25:02 main.go:47: Greeting: Hello rokety! From 10.200.177.145
2019/10/21 14:25:03 main.go:47: Greeting: Hello rokety! From 10.200.251.170
2019/10/21 14:25:04 main.go:47: Greeting: Hello rokety! From 10.200.177.145
```

3个服务端实例，只向其中2个发送了请求，通过查看3个服务端日志，确认其中有一个会在健康检查接口中返回 HealthCheckResponse_NOT_SERVING。

#### 客户端负载均衡 VS 负载均衡代理

负载均衡代理：
* 好处：客户端实现简单
* 坏处：增加延迟，增加负载均衡代理的维护成本

客户端负载均衡：
* 好处：低延迟，不需要维护负载均衡代理
* 坏处：通常只能实现简单的负载均衡策略，但是可以借助 grpclb 实现负载的负载均衡策略

关于负载均衡可以看下 grpc 的分享：[gRPC Load Balancing on Kubernetes - Jan Tattermusch, Google (Intermediate Skill Level)](https://grpc.io/docs/talks/)

本文涉及的代码和 k8s yaml 的仓库：[go-grpc-client-side-lb-example](https://github.com/yangxikun/go-grpc-client-side-lb-example)
