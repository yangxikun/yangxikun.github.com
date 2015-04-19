---
layout: post
title: "Keepalived避免无用的failover"
description: ""
category: Keepalived
tags: []
---
{% include JB/setup %}

通常情况下，当主挂掉时，从会自动切换为主。当主上的服务恢复时，则会再次抢占成为主，这里就发生了一次不必要的failover。

为了解决上述情况，可以在主的配置中vrrp_instance增加nopreempt。

#### 实验
- - -
注意：确保实验机器防火墙不会过滤掉vrrp协议的数据包。

<!--more-->

主配置：
{% highlight php linenos %}
! Configuration File for keepalived

global_defs {
   router_id LVS_DEVEL_TEST
}

vrrp_script chk_httpd {
    script "systemctl status httpd.service > /dev/null 2>&1"
    interval 2
    weight -2
}

vrrp_instance VI_1 {
    state MASTER
    interface eno16777736
    virtual_router_id 51
    priority 100
    nopreempt
    advert_int 1
    mcast_src_ip 192.168.213.128
    track_script {
        chk_httpd
    }
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.213.200
    }
}
{% endhighlight %}
从配置：
{% highlight php linenos %}
! Configuration File for keepalived

global_defs {
   router_id LVS_DEVEL_TEST
}

vrrp_script chk_httpd {
    script "systemctl status httpd.service > /dev/null 2>&1"
    interval 2
    weight -2
}

vrrp_instance VI_1 {
    state BACKUP
    interface eno16777736
    virtual_router_id 51
    priority 99
    advert_int 1
    mcast_src_ip 192.168.213.129
    track_script {
        chk_httpd
    }
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.213.200
    }
}
{% endhighlight %}

以下是`sudo tcpdump vrrp -i eno16777736`命令输出。

正常情况下：

![tcpdump vrrp](/assets/img/201504190201.png)

128挂掉，129切换为主：

![tcpdump vrrp](/assets/img/201504190202.png)

128恢复，并未抢占成为主，仍然是129对外提供服务。

但这里有一个问题，如果此时129挂掉，128不会抢占成为主！为了解决这个问题，可以在129挂掉时，将129上的Keepalived服务停止。如下修改129配置的vrrp_script：

`systemctl status httpd.service > /dev/null 2>&1;[ $? -ne 0   ] || systemctl stop keepalived.service`

现在，当129挂掉时，128能够抢占成为主，不过再恢复129时，需要重启Keepalived服务：

![tcpdump vrrp](/assets/img/201504190203.png)
