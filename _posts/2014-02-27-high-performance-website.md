---
layout: post
title: "高性能网站架构基础知识"
description: ""
category: 网站架构
tags: [网站架构]
---
{% include JB/setup %}
*学习自《构建高性能Web站点》，这里记录一些资源，以备需要时查询*

#### 缓存
- - -

1、客户端缓存：JS/CSS/图片/静态页面，通过控制http响应头部实现

相关知识：Apache [mod_headers](http://httpd.apache.org/docs/2.4/zh-cn/mod/mod_headers.html) 和 [mod_expires](http://httpd.apache.org/docs/2.4/zh-cn/mod/mod_expires.html)

2、反向代理缓存

* [Varnish](https://www.varnish-cache.org/)
* [Squid](http://www.squid-cache.org/)

3、动态内容缓存

* [memcached](http://memcached.org/)
* [redis](http://redis.io/)

<!--more-->
4、数据库缓存

5、opcode缓存

* [OPcache](http://www.php.net/manual/en/book.opcache.php)

6、WEB服务器缓存

* [mod_file_cache](http://httpd.apache.org/docs/2.4/zh-cn/mod/mod_file_cache.html)
* [mod_cache](http://httpd.apache.org/docs/2.4/zh-cn/mod/mod_cache.html)
* [mod_cache_disk](http://httpd.apache.org/docs/2.4/zh-cn/mod/mod_cache_disk.html)

#### 组件分离
- - -

* [CDN](http://www.51know.info/system_performance/cdn/cdn.html)

#### 负载均衡
- - -
* HTTP重定向
* 智能DNS解析
* 反向代理
* NAT/iptables/[IPVS(LVS)](http://www.linuxvirtualserver.org/)

#### 共享文件系统
- - -
*例如图片服务器需要将图片共享给多台WEB服务器*

* NFS

#### 内容分发与同步
- - -
*共享文件系统可能会成为制约扩展和性能的关键点。*

* 分发：SSH、SCP、SFTP、WebDAV
* 同步：rsync

分发与同步的选择：

* 文件分发需要依赖一定的应用程序逻辑，比如SCP扩展来编写代码控制文件传输，或自己开发专用的分发程序，而文件同步可以对应用程序完全透明，部署非常简单。
* 文件分发可以更好地控制触发条件，更加灵活，比如更容易实现多级复制。
* 文件同步是一种天然的异步复制操作，不阻塞Web应用程序的运行，而在Web应用中要想对文件分发实现异步复制操作，必须得借助其他支持，比如异步计算。

#### 分布式文件系统
- - -
*分发与同步需要维护许多同步脚本和配置大量分发参数，对于大规模集群不合适；缺乏整体的管理和监控。*

* [Hadoop](http://hadoop.apache.org/)

#### 数据库扩展
- - -
* 主从复制、读写分离
* [MySQL Proxy](http://dev.mysql.com/downloads/mysql-proxy/)
* 垂直分区：将不不存在关系的数据库（表）分开
* 水平分区：将同一数据表中的记录通过某一hash算法进行分离

#### 分布式计算
- - -
* [Gearman](http://gearman.org/)
* [MemcacheQ](http://memcachedb.org/memcacheq/)
* Map/Reduce：[MapReduce: Simplified Data Processing on Large Clusters ](http://research.google.com/archive/mapreduce.html)和[What is MapReduce?](http://www-01.ibm.com/software/data/infosphere/hadoop/mapreduce/)

#### 性能监控
- - -
* [Nmon](http://www.ibm.com/developerworks/cn/aix/library/analyze_aix/)
* [SNMP](http://www.infoq.com/cn/articles/snmp-system-monitor)
* [Cacti](http://www.cacti.net/)