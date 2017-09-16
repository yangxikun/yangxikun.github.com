---
layout: post
title: "Golang动手写一个Http Proxy"
description: ""
category: http
tags: []
---
{% include JB/setup %}

本文主要使用Golang实现一个可用但不够标准，支持basic authentication的http代理服务。

> 为何说不够标准，在[HTTP/1.1 RFC](https://tools.ietf.org/html/rfc2616)中，有些关于代理实现标准的条目在本文中不考虑。

#### Http Proxy是如何代理我们的请求
- - -
Http 请求的代理如下图，Http Proxy只需要将接收到的请求转发给服务器，然后把服务器的响应，转发给客户端即可。

![](/assets/img/201709160101.png)

Https 请求的代理如下图，客户端首先需要发送一个Http CONNECT请求到Http Proxy，Http Proxy建立一条TCP连接到指定的服务器，然后响应200告诉客户端连接建立完成，之后客户端就可以与服务器进行SSL握手和传输加密的Http数据了。

![](/assets/img/201709160102.png)

> 为何需要CONNECT请求？
> 因为Http Proxy不是真正的服务器，没有www.foo.com的证书，不可能以www.foo.com的身份与客户端完成SSL握手从而建立Https连接。
> 所以需要通过CONNECT请求告诉Http Proxy，让Http Proxy与服务器先建立好TCP连接，之后客户端就可以将SSL握手消息发送给Http Proxy，再由Http Proxy转发给服务器，完成SSL握手，并开始传输加密的Http数据。

<!--more-->

#### Basic Authentication
- - -
为了保护Http Proxy不被未授权的客户端使用，可以要求客户端带上认证信息。这里以[Basic Authentication](https://tools.ietf.org/html/rfc2616#page-137)为例。

客户端在与Http Proxy建立连接时，Http请求头中需要带上：
```plain
Proxy-Authorization: Basic YWxhZGRpbjpvcGVuc2VzYW1l
```

如果服务端验证通过，则正常建立连接，否则响应：
```plain
HTTP/1.1 407 Proxy Authentication Required\r\nProxy-Authenticate: Basic realm="*"
```

#### 所需要开发的功能模块
- - -
1. 连接处理
1. 从客户端请求中获取服务器连接信息
1. 基本认证
1. 请求转发

##### 连接处理

需要开发一个TCP服务器，因为HTTP服务器没法实现Https请求的代理。

Server的定义：
```go
type Server struct {
	listener   net.Listener
	addr       string
	credential string
}
```

通过Start方法启动服务，为每个客户端连接创建goroutine为其服务：
```go
// Start a proxy server
func (s *Server) Start() {
	var err error
	s.listener, err = net.Listen("tcp", s.addr)
	if err != nil {
		servLogger.Fatal(err)
	}

    if s.credential != "" {
        servLogger.Infof("use %s for auth\n", s.credential)
    }
	servLogger.Infof("proxy listen in %s, waiting for connection...\n", s.addr)

	for {
		conn, err := s.listener.Accept()
		if err != nil {
			servLogger.Error(err)
			continue
		}
		go s.newConn(conn).serve()
	}
}
```

##### 从客户端请求中获取服务器连接信息
对于http请求头的解析，参考了golang内置的http server。

getTunnelInfo用于获取：

1. 请求头
1. 服务器地址
1. 认证信息
1. 是否https请求

```go
// getClientInfo parse client request header to get some information:
func (c *conn) getTunnelInfo() (rawReqHeader bytes.Buffer, host, credential string, isHttps bool, err error) {
	tp := textproto.NewReader(c.brc)

	// First line: GET /index.html HTTP/1.0
	var requestLine string
	if requestLine, err = tp.ReadLine(); err != nil {
		return
	}

	method, requestURI, _, ok := parseRequestLine(requestLine)
	if !ok {
		err = &BadRequestError{"malformed HTTP request"}
		return
	}

	// https request
	if method == "CONNECT" {
		isHttps = true
		requestURI = "http://" + requestURI
	}

	// get remote host
	uriInfo, err := url.ParseRequestURI(requestURI)
	if err != nil {
		return
	}

	// Subsequent lines: Key: value.
	mimeHeader, err := tp.ReadMIMEHeader()
	if err != nil {
		return
	}

	credential = mimeHeader.Get("Proxy-Authorization")

	if uriInfo.Host == "" {
		host = mimeHeader.Get("Host")
	} else {
		if strings.Index(uriInfo.Host, ":") == -1 {
			host = uriInfo.Host + ":80"
		} else {
			host = uriInfo.Host
		}
	}

	// rebuild http request header
	rawReqHeader.WriteString(requestLine + "\r\n")
	for k, vs := range mimeHeader {
		for _, v := range vs {
			rawReqHeader.WriteString(fmt.Sprintf("%s: %s\r\n", k, v))
		}
	}
	rawReqHeader.WriteString("\r\n")
	return
}
```

##### 基本认证
```go
// validateCredentials parse "Basic basic-credentials" and validate it
func (s *Server) validateCredential(basicCredential string) bool {
	c := strings.Split(basicCredential, " ")
	if len(c) == 2 && strings.EqualFold(c[0], "Basic") && c[1] == s.credential {
		return true
	}
	return false
}
```

##### 请求转发
serve方法会进行Basic Authentication验证，对于http请求的代理，会把请求头转发给服务器，对于https请求的代理，则会响应200给客户端。
```go
// serve tunnel the client connection to remote host
func (c *conn) serve() {
    defer c.rwc.Close()
	rawHttpRequestHeader, remote, credential, isHttps, err := c.getTunnelInfo()
	if err != nil {
		connLogger.Error(err)
		return
	}

	if c.auth(credential) == false {
		connLogger.Error("Auth fail: " + credential)
		return
	}

	connLogger.Info("connecting to " + remote)
	remoteConn, err := net.Dial("tcp", remote)
	if err != nil {
		connLogger.Error(err)
		return
	}

	if isHttps {
		// if https, should sent 200 to client
		_, err = c.rwc.Write([]byte("HTTP/1.1 200 Connection established\r\n\r\n"))
		if err != nil {
			glog.Errorln(err)
			return
		}
	} else {
		// if not https, should sent the request header to remote
		_, err = rawHttpRequestHeader.WriteTo(remoteConn)
		if err != nil {
			connLogger.Error(err)
			return
		}
	}

	// build bidirectional-streams
	connLogger.Info("begin tunnel", c.rwc.RemoteAddr(), "<->", remote)
	c.tunnel(remoteConn)
    connLogger.Info("stop tunnel", c.rwc.RemoteAddr(), "<->", remote)
}
```

完整代码可查看：[https://github.com/yangxikun/gsproxy](https://github.com/yangxikun/gsproxy)