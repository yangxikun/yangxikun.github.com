---
layout: post
title: "Ansible搭建LNMP双机高可用"
description: ""
category: 运维
tags: [Ansible]
---
{% include JB/setup %}

最近学习了下Ansible，做了这个实验实践下。

LNMP双机高可用架构图：

![LNMP HA](/assets/img/201506140103.png)

<!--more-->

ansible playbook：[yangxikun/lnmp-ha-ansible-playbook](https://github.com/yangxikun/lnmp-ha-ansible-playbook)

参考资料：《OReilly.Ansible.Up.and.Running》和 [mogproject/ansible-playbooks/mysql-replication](https://github.com/mogproject/ansible-playbooks/tree/master/mysql-replication)

Notice:

因为有两个php-fpm对外提供服务，所以这里就涉及到会话保持的问题，如果会话是保存在php-fpm本机的，nginx文档中有讲解[Enabling Session Persistence](http://nginx.com/resources/admin-guide/load-balancer/#sticky)，不过`sticky`指令在商业版本的Nginx才有。我们可以使用ip_hash代替，但是如果一台php-fpm挂掉，那么就会失去一部分会话。另外的解决方案可以将会话保存到缓存服务器中（memcache、redis）。

运行结果：

![LNMP HA Result](/assets/img/201506140101.png)

![LNMP HA Result](/assets/img/201506140102.png)