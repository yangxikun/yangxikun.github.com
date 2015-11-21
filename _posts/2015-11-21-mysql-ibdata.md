---
layout: post
title: "MySQL ibdata 存储空间的回收"
description: ""
category: MySQL
tags: [InnoDB]
---
{% include JB/setup %}

### 前言
- - -
在MySQL <= 5.6.5，[innodb\_file\_per\_table](http://dev.mysql.com/doc/refman/5.6/en/innodb-parameters.html#sysvar_innodb_file_per_table)默认为0，即InnoDB表的数据都会存储在共享表空间ibdata中，除此之外ibdata还存储着数据字典、双写缓冲区、undo log等。

当innodb\_file\_per\_table为0时，ibdata会不断增大，有时会导致磁盘空间不足。通常是InnoDB表的数据导致的，undo log是次要原因。
因为undo log的增加通常是在事务较为繁忙的时候，且事务中做了大量的更新操作，但是undo log占用的空间却可以被重用。InnoDB的purge线程就是负责清理不需要的undo log空间以供其他的undo log使用。

那么为何InnoDB表的数据会成为ibdata增大的主要原因？因为InnoDB表的数据被delete之后的空间是无法被InnoDB重用的，需要人为干预处理= =

<!--more-->

### 实验
- - -
实验条件：MySQL 5.5.9、innodb\_file\_per\_table = 0、新初始化的MySQL数据库目录

一、创建数据库t、数据表t1、用于向t1表插入数据的存储过程load\_t1

{% highlight sql linenos %}
mysql> create database t;
Query OK, 1 row affected (0.01 sec)

mysql> use t;
Database changed
mysql> create table t1 (
    -> id int not null auto_increment,
    -> content varchar(7000),
    -> primary key (id))engine=InnoDB;
Query OK, 0 rows affected (0.00 sec)

mysql> delimiter //
mysql> create procedure load_t1(count int unsigned)
    -> begin
    -> declare s int unsigned default 1;
    -> declare c varchar(7000) default repeat('A', 7000);
    -> while s <= count do
    -> insert into t1 select NULL, c;
    -> set s = s+1;
    -> end while;
    -> end;
    -> //
Query OK, 0 rows affected (0.00 sec)

mysql> delimiter ;
{% endhighlight %}

MySQL数据目录：

![mysql](/assets/img/201511210101.png)

二、分两次插入500行数据

![mysql](/assets/img/201511210102.png)

在第一次插入500行数据时由于共享表空间剩余不多，所以ibdata此时增长了8M：

![mysql](/assets/img/201511210103.png)

在第二次插入500行数据时由于共享表空间剩余足够多，所以ibdata没有增长。

三、删除这1000行数据
发现从information\_schema.TABLES获取的信息没变：

![mysql](/assets/img/201511210104.png)

ibdata的大小也没有改变。

四、插入500行数据
DATA FREE和DATA LENGTH都变大了：

![mysql](/assets/img/201511210105.png)

ibdata的大小也增大了(35651584-27262976)/(1024\*1024)=8M：

![mysql](/assets/img/201511210106.png)

插入的这500行数据使得DATA FREE变小了，也即共享表空间的空闲空间缩小了，从而需要再次自增，DATA FREE也增大了。

> 注：在独立表空间下，进行如上实验，也会发现delete掉的空间无法被重用，ibd文件随着insert不断增长，但执行truncate table的话能够回收掉表占用的空间，并且返还给文件系统。而在共享表空间下执行truncate table能够回收空间（返还给DATA FREE），但不会返还给文件系统。（drop跟truncate操作类似）

### 那么要如何将“浪费”的空间回收来利用呢？
通过[OPTIMIZE TABLE](http://dev.mysql.com/doc/refman/5.5/en/optimize-table.html)能否对空间进行回收？OPTIMIZE TABLE简介：

> Reorganizes the physical storage of table data and associated index data, to reduce storage space and improve I/O efficiency when accessing the table. **The exact changes made to each table depend on the storage engine used by that table.** This statement does not work with views. 

> Use OPTIMIZE TABLE in these cases, depending on the type of table: 
>> After doing substantial insert, update, or delete operations on an InnoDB table that has its own .ibd file because it was created with the **innodb\_file\_per\_table option enabled**. The table and indexes are reorganized, and **disk space can be reclaimed for use by the operating system**. 

从文档中可以看出在使用独立表空间的情况下，OPTIMIZE TABLE才能够回收“浪费”的空间。
那么在共享表空间下执行OPTIMIZE TABLE会是什么样子的呢？继续之前的实验，执行`OPTIMIZE TABLE t1;`

![mysql](/assets/img/201511210107.png)

DATA LENGTH减少的空间加入到了DATA FREE中了，再插入两次500行数据：

![mysql](/assets/img/201511210108.png)

DATA FREE减少，DATA LENGTH增加，但DATA FREE还未小到需要ibdata自增，所以此时ibdata的大小仍是35651584。

那么t1表回收的空间能否被其它表重用呢？继续实验，新建一张t2表，删除t1表的内容并做OPTIMIZE，向t2表插入数据：

![mysql](/assets/img/201511210109.png)

![mysql](/assets/img/201511210110.png)

![mysql](/assets/img/201511210111.png)

可以看到t2表的DATA LENGTH增加，DATA FREE减少，但DATA FREE还未小到需要ibdata自增，所以此时ibdata的大小仍是35651584。
**说明在共享表空间下，对表做OPTIMIZE TABLE能够使之前因delete浪费的空间回收来重用，但是ibdata并不会收缩。**

> 注：在共享表空间执行完OPTIMIZE后通常会使ibdata增长！！上面实验ibdata没增长是因为我每次delete的都是全表数据，如果我只delete一部分数据的话就会导致ibdata增长了！
> 下图对新建的t1表做了三次插入一次删除操作之后进行OPTIMIZE操作，可以发现总的DATA LENGTH+DATA FREE是增长的！查看ibdata的大小也发现增长了。这是因为在共享表空间下，执行OPTIMIZE后，重新组织的连续的数据、索引页是被追加到ibdata中的。参阅：[Howto: Clean a mysql InnoDB storage engine?](http://stackoverflow.com/questions/3927690/howto-clean-a-mysql-innodb-storage-engine/4056261)

![mysql](/assets/img/201511210112.png)

那么如果我想让回收的空间返还给文件系统呢？根据OPTIMIZE的文档，可以通过使用独立表空间来解决，再做次实验= =

实验条件：MySQL 5.5.9、innodb\_file\_per\_table = 1、新初始化的MySQL数据库目录

一、创建数据库t、数据表t1

![mysql](/assets/img/201511210113.png)

逐步插入数据，发现DATA FREE始终是4194304(4MB)，不知为啥= =，DATA LENGTH是增长的，t1.ibd则是以最小1M的大小增长（98304->12582912->16777216->17825792->18874368）。

二、删除t1数据，再做OPTIMIZE

![mysql](/assets/img/201511210114.png)

t1.ibd收缩了！

![mysql](/assets/img/201511210115.png)

**说明在独立表空间下，对表OPTIMIZE TABLE能够使得InnoDB引擎表因delete“浪费掉的空间”得以回收并且返还给文件系统**

> 注：上面实验如果只删除一部分t1数据后再执行OPTIMIZE操作，t1.ibd页是会收缩的。

### 总结
* 建议使用独立表空间，即innodb\_file\_per\_table设置为1（MySQL>= 5.6.6默认开启）
* 设置[innodb\_purge\_threads](https://dev.mysql.com/doc/refman/5.5/en/innodb-improved-purge-scheduling.html)为1可以提升点性能
* 最终极的回收空间方法就是通过mysqldump进行逻辑导出后再导入了= =参阅：[Howto: Clean a mysql InnoDB storage engine?](http://stackoverflow.com/questions/3927690/howto-clean-a-mysql-innodb-storage-engine/4056261)
