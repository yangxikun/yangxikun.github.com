---
layout: post
title: "MySQL 大表处理"
description: "MySQL 大表处理"
category: MySQL
tags: []
---
{% include JB/setup %}

#### 背景

业务有一张表现在有1亿多条记录，表大小500G，由于对历史数据不会再访问，可以将历史数据进行归档。

如果大表的数据都是会被访问的可以考虑使用 MySQL 的分区表，但 MySQL 的分区表功能有些限制，可参考：[Restrictions and Limitations on Partitioning](https://dev.mysql.com/doc/refman/5.7/en/partitioning-limitations.html)。如果不方便使用 MySQL 的分区表的话，可以考虑在业务上实现。

#### 归档

Percona Toolkit 提供了一个 perl 编写的归档工具：[pt-archiver](https://www.percona.com/doc/percona-toolkit/LATEST/pt-archiver.html)

如果采用归档到文件的方式，该归档工具的工作流程类似下面的伪代码：

> 以下流程是笔者对 pt-archiver 与 MySQL 的连接进行抓包分析出来的。

```text
set autocommit = 0
chunk = select archive rows limit n // --limit
count = 0
for row in chunk:
    archive one row
    delete one row
    count++
    if count == txn-size: // --txn-size
        count = 0
        commit
    if chunk is empty:
        chunk = select archive rows limit n // --limit
```

* --txn-size：太小会出现非常多的小事务，增加 MySQL 负载（比如当innodb_flush_log_at_trx_commit=1时，每一个事务的commit都需要进行flush操作），太大会导致事务执行时间过长，增加出现锁等待和死锁的概率。
* --limit：在 innodb 表上，如果没有使用 --for-update 参数的话，则 select 的时候不会跟其他事务产生锁竞争。

笔者采用的命令参数如下：

```text
# 先导出数据，不删除表里的数据
pt-archiver --charset utf8 --source h=${MYSQL_HOST},D=${MYSQL_DB},t=${MYSQL_TABLE} --user ${MYSQL_USER} --ask-pass --file '%Y-%m-%d-%D.%t' --progress 1000 --where 'created_at < "2019-01-01 00:00:00"' --txn-size 100 --limit 1000 --no-delete
# 确认导出的数据没有问题，删除表中的数据
pt-archiver --charset utf8 --source h=${MYSQL_HOST},D=${MYSQL_DB},t=${MYSQL_TABLE} --user ${MYSQL_USER} --ask-pass --progress 1000 --where 'created_at < "2019-01-01 00:00:00"' --txn-size 100 --limit 1000 --nosafe-auto-increment --purge
```

* 默认情况下，pt-archiver的查找会强制使用主键索引，虽然 match_logs 表上有 created_at 字段的索引，但是我们不能使用该索引，因为它不是唯一索引，如果使用 created_at 字段的索引会导致导出来的数据少了，这跟 pt-archiver 如何获取下一页数据有关
* 当pt-archiver适合使用主键索引时，默认情况下，不会归档最后一条记录（即自增主键最大的记录，这是出于安全考虑，因为MySQL自增列的最大值在5.7中是存储在内存中的，没有持久化到磁盘，重启后会通过 SELECT MAX(ai_col) FROM table_name FOR UPDATE; 将获取到的值+1作为下一个可用的自增值），如果归档最后一条记录没有影响，可以添加--nosafe-auto-increment参数
* 归档完数据后，还需要执行下`OPTIMIZE TABLE ${MYSQL_TABLE}`，以回收空间，详细原因可参考文章：[MySQL ibdata 存储空间的回收](http://yangxikun.com/mysql/2015/11/21/mysql-ibdata.html)