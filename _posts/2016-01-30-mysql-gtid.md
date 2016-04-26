---
layout: post
title: "MySQL Gtid复制方案学习"
description: ""
category: 
tags: []
---
{% include JB/setup %}

MySQL从5.6开始出了新的主从复制解决方案：[Replication with Global Transaction Identifiers](https://dev.mysql.com/doc/refman/5.7/en/replication-gtids.html "Replication with Global Transaction Identifiers")。

GTID解决的问题：

* 在整个复制集群中能够唯一的标识一个事务
* 更方便的实效转移
* 确保同一个事务只会被执行一次

GTID的限制：

* 无法使用`CREATE TABLE ... SELECT statements`语句
* 无法在事务中对非事务存储引擎进行更新
* 无法在事务中使用`CREATE TEMPORARY TABLE`
* 具体可参考：[Restrictions on Replication with GTIDs](https://dev.mysql.com/doc/refman/5.7/en/replication-gtids-restrictions.html)

<!--more-->

#### GTID相关概念

下文主要围绕与复制相关的概念展开，欲详细了解的可查看官方文档，并参考了如下博文：

* [MySQL5.6 ID（GTID） principle （ a global transaction ）](http://db.codes9.com/mysql/mysql5-6-id%EF%BC%88gtid%EF%BC%89-principle-%EF%BC%88-a-global-transaction-%EF%BC%89/)
* [MySQL5.6 The global transaction ID（GTID） principle （ two ）](http://db.codes9.com/mysql/mysql5-6-the-global-transaction-id%EF%BC%88gtid%EF%BC%89-principle-%EF%BC%88-two-%EF%BC%89/)
* [MySQL5.6 The global transaction ID（GTID） principle （ three ）](http://db.codes9.com/mysql/mysql5-6-the-global-transaction-id%EF%BC%88gtid%EF%BC%89-principle-%EF%BC%88-three-%EF%BC%89/)

- - -
GTID表现形式：`source_id:transaction_id`，其中source_id为一个全局的系统变量[server_uuid](https://dev.mysql.com/doc/refman/5.7/en/replication-options.html#sysvar_server_uuid)，transaction_id为一个从1开始自增的数值。

GTID Sets：一种聚合GTID的数据结构，表现形式：`array[sidno => link_list[Interval]]`，如下图：

![](/assets/img/201601300101.gif)

sidno是数组索引，Interval用来存放一组事务的区间，sidno与server_uuid也存在一对一的关联。

假如server_uuid:283f1ec3-c2a9-11e5-8af6-000c2930e841对应的sidno为1，那么在GTID Sets中的表现形式：`[|sidno:1|->interval(1, 143)]`。

该数据结构的作用：

1. 系统变量gtid_executed、gtid_purged的值就是使用Gtid Sets表示的
1. 能够用于快速判断一个事务是否已经被执行过了
1. GTID_SUBSET()、GTID_SUBTRACT()等函数要求以Gtid Sets作为输入

mysql.gtid_executed：不管是否开启了二进制日志，都会将执行过的事务的GTID保存到该表中（如果开启了二进制日志，MySQL是先写GTID到二进制日志中，后写入mysql.gtid_executed）

与主从复制相关的几个GTID系统变量：

* gtid_executed：保存着在server中所有执行过的事务的GTID和`SET gtid_purged`设置的GTID。
* gtid_purged：保存着在二进制日志中被清除掉了的GTID，它是gtid_executed的一个子集。

MySQL启动时对gtid_executed和gtid_purged的初始化（见[binlog_gtid_simple_recovery](https://dev.mysql.com/doc/refman/5.7/en/replication-options-gtids.html#sysvar_binlog_gtid_simple_recovery)）：

* gtid_executed：从最新的二进制日志文件往回查找，直到找到第一个Previous_gtids_log_event，将Previous_gtids_log_event、Gtid_log_events、mysql.gtid_executed中的Gtid这三个GTID Sets做UNION，其中Previous_gtids_log_event、Gtid_log_events的UNION值赋值给gtids_in_binlog。
* gtid_purged：从最老的二进制日志文件往前查找，直到找到第一个非空的Previous_gtids_log_event（赋值给tmp_gtid_purged）或者至少有一个Gtid_log_event（tmp_gtid_purged赋值为空）。`gtids_in_binlog - tmp_gtid_purged = gtids_in_binlog_not_purged`，最后`gtid_purged = gtid_executed - gtids_in_binlog_not_purged`。

通过`mysqlbinlog binlogFIle --base64-output=DECODE-ROWS`命令可查看到：

Previous_gtids_log_event的样子如下：

![](/assets/img/201601300102.png)
Gtid_log_event的样子如下：

![](/assets/img/201601300103.png)

slave请求master时发送的请求：`Request = { server_id, binlog_name, binlog_offset, gtids_executed }  `

在GTID模式下启用了`MASTER_AUTO_POSITION `的话，请求中的`binlog_name`和`binlog_offset`将是空的，master是根据请求中的`gtids_executed`返回binlog。

#### 1主1从配置实践
- - -
数据库版本：MySQL 5.7.10

mater配置如下：
{% highlight c linenos %}
log-bin=master  //主从同步需要开启binlog
server-id=1     //虽然mysql会自动产生server_uuid，但还是需要配置它= =

gtid-mode=ON    //开启GTID模式
enforce-gtid-consistency=ON //开启GTID的限制，防止出现数据不一致
{% endhighlight %}
slave配置如下：
{% highlight c linenos %}
log-bin=slave
relay_log=slave-2
server-id=2
read_only=1
log_slave_updates=1 //能够成为其他slave的master
skip-slave-start=1

gtid-mode=ON
enforce-gtid-consistency=ON
{% endhighlight %}

分别启动master和slave（我这里用的是MySQL的Docker镜像），在master上创建用于复制的账号，直接进行CHANGE MASTER TO是否可行呢？

不行，relay sql时报错了：

![](/assets/img/201601300104.png)

relay sql执行了`CREATE DATABASE mysql`，当前slave已经有mysql数据库了，所以就抛出错误了。

先看看当前的gtids_executed：

![](/assets/img/201601300105.png)
其实gtids_executed与`SHOW SLAVE STATUS\G`中的Executed_Gtid_Set是同一个值：

![](/assets/img/201601300106.png)
从Retrieved_Gtid_Set可看到接收到的GTID Sets，master的server_uuid为`46cda27d-c601-11e5-9f9b-0242ac110002`，而通过`SHOW global variables LIKE 'server_uuid'`可查到slave的server_uuid为`da4fa606-c602-11e5-adb1-0242ac110003`。

说明当前slave所执行过的事务都是slave上产生的，relay sql出错是在执行master传输过来的binlog产生的，我猜测是master在接收到slave请求参数中的gtid_executed发现自己的二进制文件中并不存在gtid为`da4fa606-c602-11e5-adb1-0242ac110003:x`的事务就从二进制日志中存在的第一条事务开始发送回给slave。

查看下master上第一条事务的binlog（查看第一个binlog文件的开头）：

![](/assets/img/201601300107.png)

要解决relay sql的错误，应该是让slave从master获取合适的binlog，所以需要设置好slave合适的gtid_executed，让master发送合适的binlog给slave。因为master是一台全新的服务器，没有任何“用户”数据，所以把slave上的gtid_executed设置为`46cda27d-c601-11e5-9f9b-0242ac110002:1-140`就可以同步master接下来执行的事务了。

但是我们无法直接修改gtids_executed的值，如果直接修改，会报错：`Variable 'gtid_executed' is a read only variable`。

查看官方文档[gtid_executed的说明](https://dev.mysql.com/doc/refman/5.7/en/replication-options-gtids.html#sysvar_gtid_executed)，讲得有点看不太懂，但至少知道`SET gtid_purged`是会影响到gtid_executed的值的。

查看官方文档[gtid_purged的说明](https://dev.mysql.com/doc/refman/5.7/en/replication-options-gtids.html#sysvar_gtid_purged)，其中指出：如果gtid_executed是空的，就能够更新`gtid_purged`的值。（PS：不知是我英文不好，还是官网的文档描述确实有问题，有些说明感觉不太对= =）

通过`RESET MASTER`能够更新gtid_executed的值为空，再设置gtid_purged的值看看：

![](/assets/img/201601300108.png)
如果想重新初始化relay log的话，可以在上图中执行`RESET SLAVE`，这样的话需要重新执行`CHANGE MASTER TO`。

gtid_executed的值同样被更新为`46cda27d-c601-11e5-9f9b-0242ac110002:1-140`了，重新执行`START SLAVE`，查看slave状态：

{% highlight c linenos %}
Slave_IO_Running              | Yes
Slave_SQL_Running             | Yes
Slave_SQL_Running_State       | Slave has read all relay log; waiting for more updates
Retrieved_Gtid_Set            | 46cda27d-c601-11e5-9f9b-0242ac110002:1-140
Executed_Gtid_Set             | 46cda27d-c601-11e5-9f9b-0242ac110002:1-140
{% endhighlight %}
一切正常，现在在master执行一些语句（创建了一个数据库、一张表），查看slave状态：

{% highlight c linenos %}
Slave_IO_Running              | Yes
Slave_SQL_Running             | Yes
Slave_SQL_Running_State       | Slave has read all relay log; waiting for more updates
Retrieved_Gtid_Set            | 46cda27d-c601-11e5-9f9b-0242ac110002:1-142
Executed_Gtid_Set             | 46cda27d-c601-11e5-9f9b-0242ac110002:1-142
{% endhighlight %}
再查看slave上确实已经同步了新建的数据库、数据表。

再做一个测试，停止slave，在master上的test_repl.users表中插入两条数据，再启动slave，由于slave设置了`skip-slave-start=1`，所以还需要手动执行`START SLAVE`，执行后查看slave状态是正常的，也成功同步了两条新插入的数据。

#### 1主2从，模拟master挂掉，其中一台slave提升为master，另外一台slave切换到新的master上同步
- - -
新启动3台MySQL实例，1主2从模式，配置好它们之间的复制（master、slave1先创建用于复制的账号）。

在master上新建test_repl数据库，users表，插入一条数据，master上的状态：

![](/assets/img/201601300109.png)

查看两台slave状态，同步正常：

{% highlight c linenos %}
slave1:
Slave_IO_Running              | Yes
Slave_SQL_Running             | Yes
Slave_SQL_Running_State       | Slave has read all relay log; waiting for more updates
Retrieved_Gtid_Set            | a7f0745e-c764-11e5-b21d-0242ac110002:141-143
Executed_Gtid_Set             | a7f0745e-c764-11e5-b21d-0242ac110002:1-143

mysql root@172.17.0.3:(none)> SELECT * FROM test_repl.users;
+------+--------+
|   id | nick   |
|------+--------|
|    1 | repl1  |
+------+--------+
1 row in set

slave2:
Slave_IO_Running              | Yes
Slave_SQL_Running             | Yes
Slave_SQL_Running_State       | Slave has read all relay log; waiting for more updates
Retrieved_Gtid_Set            | a7f0745e-c764-11e5-b21d-0242ac110002:141-143
Executed_Gtid_Set             | a7f0745e-c764-11e5-b21d-0242ac110002:1-143

mysql root@172.17.0.4:(none)> SELECT * FROM test_repl.users;
+------+--------+
|   id | nick   |
|------+--------|
|    1 | repl1  |
+------+--------+
{% endhighlight %}
接下来执行如下步骤：

1、先停下slave2的复制

2、在master上再插入一条数据

3、停掉master，停止slave1上的复制，更改slave1的read_only的值为0

现在slave1的状态：

{% highlight c linenos %}
Slave_IO_Running              | No
Slave_SQL_Running             | No
Last_IO_Error                 | error reconnecting to master 'repl@172.17.0.2:3306' - retry-time: 60  retries: 1
Retrieved_Gtid_Set            | a7f0745e-c764-11e5-b21d-0242ac110002:141-144
Executed_Gtid_Set             | a7f0745e-c764-11e5-b21d-0242ac110002:1-144

mysql root@172.17.0.3:(none)> SET GLOBAL read_only = OFF;
Query OK, 0 rows affected
Time: 0.006s
mysql root@172.17.0.3:(none)> SHOW VARIABLES LIKE 'read_only'
+-----------------+---------+
| Variable_name   | Value   |
|-----------------+---------|
| read_only       | OFF     |
+-----------------+---------+
1 row in set
Time: 0.010s
{% endhighlight %}
4、在slave2上执行CHANGE MASTER TO切换到新的master（slave1）上，重新启动复制
现在slave2的状态：

{% highlight c linenos %}
Slave_IO_Running              | Yes
Slave_SQL_Running             | Yes
Slave_SQL_Running_State       | Slave has read all relay log; waiting for more updates
Retrieved_Gtid_Set            | a7f0745e-c764-11e5-b21d-0242ac110002:144
Executed_Gtid_Set             | a7f0745e-c764-11e5-b21d-0242ac110002:1-144

mysql root@172.17.0.4:(none)> SELECT * FROM test_repl.users;
+------+--------+
|   id | nick   |
|------+--------|
|    1 | repl1  |
|    2 | repl2  |
+------+--------+
2 rows in set
Time: 0.007s
{% endhighlight %}
5、在新的master（slave1）上插入数据，查看slave1的状态：

![](/assets/img/201601300110.png)
6、查看此时slave2的状态：

{% highlight c linenos %}
Retrieved_Gtid_Set            | a7f0745e-c764-11e5-b21d-0242ac110002:144,
b17fbdfc-c764-11e5-b238-0242ac110003:1
Executed_Gtid_Set             | a7f0745e-c764-11e5-b21d-0242ac110002:1-144,
b17fbdfc-c764-11e5-b238-0242ac110003:1
Slave_SQL_Running_State       | Slave has read all relay log; waiting for more updates
Slave_IO_Running              | Yes
Slave_SQL_Running             | Yes

mysql root@172.17.0.4:(none)> SELECT * FROM test_repl.users;
+------+--------+
|   id | nick   |
|------+--------|
|    1 | repl1  |
|    2 | repl2  |
|    3 | repl3  |
+------+--------+
3 rows in set
Time: 0.007s
{% endhighlight %}
复制一切正常，对比binlogFile+offset的复制方案，slave2不需要知道slave1的binlogFile+offset，失效转移就更简单了。从Executed_Gtid_Set也能够看出slave上执行的事务分别来自哪台master。