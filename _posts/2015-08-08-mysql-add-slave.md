---
layout: post
title: "MySQL 主从复制添加slave"
description: ""
category: mysql
tags: [MySQL复制]
---
{% include JB/setup %}

在已有的MySQL主从复制下添加slave的方法：

方法一：

1、在已有的slave上，执行`stop slave io_thread;`，等待Slave_open_temp_tables为0时，执行`stop slave sql_thread`；

2、然后执行`flush tables with read lock;`和`flush logs`；

3、拷贝整个datadir（bin-log、relay-log等信息也要在这里面）内容到new-slave的datadir下；

4、在new-slave上启动mysql后执行`start slave;`即可。

<!--more-->

方法二：

方法一需要拷贝较多数据，我们也可以只拷贝需要进行复制的数据库。

1、在已有的slave上，执行`stop slave io_thread;`，等待Slave_open_temp_tables为0时，执行`stop slave sql_thread`；

2、然后执行`flush tables with read lock;`和`flush logs`；

3、执行`show slave status\G;`获取replication position；

4、拷贝需要进行复制的数据库和ibdata1、ib_logfile1、ib_logfile0文件到new-slave的datadir下；

5、执行change master。

此方法可以采用Percona XtraBackup实现，相关信息：

* [How to setup a slave for replication in 6 simple steps with Percona XtraBackup](https://www.percona.com/doc/percona-xtrabackup/2.1/howtos/setting_up_replication.html#replication-howto)
* [Taking Backups in Replication Environments](https://www.percona.com/doc/percona-xtrabackup/2.1/innobackupex/replication_ibk.html)
* [about-apply-log](https://www.percona.com/forums/questions-discussions/percona-xtrabackup/8775-about-apply-log)
