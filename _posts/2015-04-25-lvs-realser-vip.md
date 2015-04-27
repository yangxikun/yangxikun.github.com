---
layout: post
title: "LVS DR/TUN real server VIP配置"
description: ""
category: 
tags: []
---
{% include JB/setup %}

在LVS的DR/TUN模式中，需要在real server上配置与LVS Director相同的VIP，这样real server才能接受和处理LVS Director转发过来的数据包。

但是这样一个网络内就有多台主机拥有VIP，当有数据包的目的IP地址是VIP时，该传输给那台主机？显然我们希望是由LVS Director接收。

那么就相当于real server需要对其他主机隐藏自己拥有VIP的事实，这可以通过ARP机制实现：当real server接收到目的IP地址是VIP的ARP广播请求包时，不做出回应。

现在的Linux内核支持通过两个参数配置网卡接口对ARP的请求和响应做出不同的处理：

1. arp_announce: 在ARP请求数据包中该包含哪个源IP地址定义不同的限制级别。
1. arp_ignore: 对ARP请求包中目的IP地址定义不同的限制级别。

<!--more-->

{% highlight php linenos %}
arp_announce - INTEGER
        Define different restriction levels for announcing the local
        source IP address from IP packets in ARP requests sent on
        interface:
        0 - (default) Use any local address, configured on any interface
        1 - Try to avoid local addresses that are not in the target's
        subnet for this interface. This mode is useful when target
        hosts reachable via this interface require the source IP
        address in ARP requests to be part of their logical network
        configured on the receiving interface. When we generate the
        request we will check all our subnets that include the
        target IP and will preserve the source address if it is from
        such subnet. If there is no such subnet we select source
        address according to the rules for level 2.
        2 - Always use the best local address for this target.
        In this mode we ignore the source address in the IP packet
        and try to select local address that we prefer for talks with
        the target host. Such local address is selected by looking
        for primary IP addresses on all our subnets on the outgoing
        interface that include the target IP address. If no suitable
        local address is found we select the first local address
        we have on the outgoing interface or on all other interfaces,
        with the hope we will receive reply for our request and
        even sometimes no matter the source IP address we announce.

        The max value from conf/{all,interface}/arp_announce is used.

        Increasing the restriction level gives more chance for
        receiving answer from the resolved target while decreasing
        the level announces more valid sender's information.

arp_ignore - INTEGER
        Define different modes for sending replies in response to
        received ARP requests that resolve local target IP addresses:
        0 - (default): reply for any local target IP address, configured
        on any interface
        1 - reply only if the target IP address is local address
        configured on the incoming interface
        2 - reply only if the target IP address is local address
        configured on the incoming interface and both with the
        sender's IP address are part from same subnet on this interface
        3 - do not reply for local addresses configured with scope host,
        only resolutions for global and link addresses are replied
        4-7 - reserved
        8 - do not reply for all local addresses

        The max value from conf/{all,interface}/arp_ignore is used
        when ARP request is received on the {interface}
{% endhighlight %}

上面这段英文文档只能看个半懂，网上其他博客的翻译也是看不懂。。。

real server的配置：

1、将VIP设置在lo网卡接口上，并且子网掩码必须与其他网卡的子网掩码不同，否则其他机器将无法与此机器通信

ifconfig输出：

![ifconfig output](/assets/img/201504250101.png)


执行命令：`sudo ifconfig lo:0 192.168.213.200 netmask 255.255.255.255 up`

![ifconfig output](/assets/img/201504250102.png)

如果命令为：`sudo ifconfig lo:0 192.168.213.200 netmask 255.255.255.0 up`，那么其他机器将无法ping通此主机。具体为什么我也没查到原因，在stackoverflow提了个问题[netmask of VIP on loopback interface confuse](http://stackoverflow.com/questions/29839293/netmask-of-vip-on-loopback-interface-confuse)，但没人回答。。。

2、设置arp_ignore的值为1

执行命令：`echo "1" > /proc/sys/net/ipv4/conf/all/arp_ignore`

即当eno16777736接口收到对目的IP地址为VIP的ARP广播请求包时，不做出响应。默认情况下，arp_ignore的值为0，即使eno16777736接口上没有VIP，但是当前主机其他接口上有VIP，则做出ARP响应。

看网上好多博文都是要设置arp_announce为2，但自己实验时发现只需要设置arp_ignore为1就可以了。
