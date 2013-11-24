---
layout: post
title: "Session和Cookie小结以及PHP单点登陆的实现"
description: ""
category: PHP
tags: [PHP应用]
---
{% include JB/setup %}

### Cookie

由于HTTP协议设计原则是无状态的*\(我的理解是:http的一个请求状态与下一个或上一个请求状态无关\)*,但是近年来出现了种种需求，其中cookie的作用就是为了解决HTTP协议无状态的缺陷所作出的努力。后来出现的session机制则又是一种在客户端与服务器之间保持状态的解决方案。

Cookie的属性：name(名字)；value(值)；expire(过期时间)；path(路径)；domain(域)；secure(安全)；httponly(仅http)

domain(域)：低级的域可以获取到高级的域的cookie，但需要注意，例如`t.domaina.com`可以获得`.domaina.com`的cookie，而无法获得`domaina.com`的cookie。也就是说一个诸如这样的域名`domaina.com`，它可以设置域为`domain.com`和`.domain.com`的cookie，且只有`.domain.com`的cookie在子域名下可见。需要注意的是在PHP中，当你使用`setcookie('ck', 'testSetdomainacookie', 0, '/', 'domaina.com');`和` setcookie('ck', 'testSetdomainacookie', 0, '/', '.domaina.com');`效果是一样的，设置的cookie域都是`.domaina.com`。而使用`setcookie('ck', 'testSetdomainacookie', 0, '/')`设置的cookie域为`domaina.com`。对于子域名t.domaina.com还可以设置域为`.domain.com`的cookie，也即子域名下可以设置父域名可见的cookie。

path(路径)：同domain一样,子目录可以访问上级目录的cookie,但是使用setcookie函数时,这个path必须是在你所访问的目录下,例如直接在web服务器根目录执行`setcookie('localhost',520,3600+time(),'/pathcookie')`是无法设置cookie的,但php不会报错，而如果在web服务器根目录下新建一个名为pathcookie的目录，
并在此目录下执行`setcookie('localhost',520,3600+time(),'/pathcookie')`才能设置cookie,另外在web服务器根目录下新建的pathcookie下执行`setookie('localhost',520,3600+time(),'/')`也是可以的。以上说明了在子目录的setcookie可以将cookie的path设置为父目录，而父目录中setcookie无法设置path为子目录!

secure(安全)：只能通过HTTPS进行传输。

httponly(仅http)：只能通过http协议访问cookie，能够防止js访问cookie，需要浏览器支持。

### Session

Session即会话机制,性质:名称；值；保存路径；保存方式；过期时间

在php.ini中可以设置许多与session相关的信息：具体见手册[关于php.ini中设置session](http://www.php.net/manual/zh/session.configuration.php#ini.session.upload-progress.enabled)

其中session.save_handler和session.save_path是想对应的，例如handler为file时，path为目录；handler为memcached时,path为一个tcp链接。

可以看到在那么多设置选项中有好几个是设置cookie的，其实客户端的cookies是为了能够标识用户在服务器端拥有哪个session，从而实现会话.在客户端这个cookie的名称即为session的名称，cookie的值即为session_id，这个session_id可以唯一标识出服务器上某个session。

再了解下session的垃圾回收机制：

在php.ini中可以看到如下关于设置session的垃圾回收机制：

session.gc_probability 与 session.gc_divisor 合起来用来管理 gc（garbage collection 垃圾回收）进程启动的概率。默认为 1。

session.gc_divisor 与 session.gc_probability 合起来定义了在每个会话初始化时启动 gc（garbage collection 垃圾回收）进程的概率。此概率用 gc_probability/gc_divisor 计算得来。例如 1/100 意味着在每个请求中有 1% 的概率启动 gc 进程。session.gc_divisor 默认为 100。

session.gc_maxlifetime 指定过了多少秒之后数据就会被视为“垃圾”并被清除。这里并不会说立即清除，而是只有当gc启动时，才会清楚。

由于PHP的工作机制，它并没有一个daemon线程，来定时地扫描session信息并判断其是否失效。当一个有效请求发生时，PHP会根据全局变量 session.gc_probability/session.gc_divisor（同样可以通过php.ini或者ini_set()函数来修改） 的值，在初始化session后(即session_start()),来决定是否启动一个GC（Garbage Collector）。

GC的工作，就是扫描所有的session信息， 用当前时间减去session的最后修改时间（modified date），同session.gc_maxlifetime参数进行比较，如果生存时间已经超过gc_maxlifetime，就把该session删 除。

### 实现单点登陆

在github上找到了sso的实例[Simple Single Sign-On for PHP](https://github.com/jasny/SSO)。花了一天的时间研究了下源码。基本清楚实现流程。当然还存在其他的SSO解决方案，作者在README中说明了这个解决方案简单易用，且配合AJAX也能工作完好。在README中，作者已经介绍了实现流程，我看了还是不太懂，看了下源码加上实际操作才明白其中的道理。

这里我指出解决方案中关键的几点：

* 真正与用户相关的session是存储在SSO上的；

* client存储了session_token和PHPSESSION，session_token为broker的cookie，PHPSESSION为SSO的cookie；

* 当broker请求SSO绑定自己的时候，在SSO上，以`sess_`+对broker和token等的加密作为软连接到真正与用户相关的session文件；

* broker通过CURL发送包含session_token的请求给SSO，用以验证用户是否登陆；

* 在使用AJAX的网站中，由于js无法跨站请求（jsonp可以），解决方案中是通过`<img src="请求地址">`
的方式来实现在client设置SSO的cookie，在不使用AJAX的情况下，是通过重定向来实现的。

还有两点需要注意：

* 凡是需要登陆用户才能执行的操作，broker都得通过CURL发送请求到SSO用以验证用户是否仍处于登陆状态，所以如果broker与SSO之间通信时间较长，可能会影响到用户的使用。

* 由于在SSO上为broker生成的软连接不会被gc回收，我们得自己实现一个回收程序。