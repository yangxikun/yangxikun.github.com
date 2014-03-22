---
layout: post
title: "MySQL查询优化学习"
description: ""
category: MySQL
tags: [MySQL查询优化]
---
{% include JB/setup %}

*学习自《高性能MySQL 第三版》和网络资源*

一张图了解MySQL执行SQL语句过程：

![](/assets/img/201403200101.png)

#### 是否向数据库请求了不需要的数据
- - -
* 查询了不需要的记录
* 多表关联时返回全部列
* 总是取出全部列

解决方案：1、使用 LIMIT；2、避免使用 \*，从而有机会使用索引覆盖查询。

<!--more-->

#### 增加合适的索引
- - -
使用EXPLAIN分析查询，type 列反应了查询的类型。从全表扫描、索引扫描、范围扫描、唯一索引扫描、常数引用等，速度由慢到快，扫描的行数由大到小。

一般MySQL能够使用如下三种方式应用WHERE条件，从好到坏依次为：

* 在索引中使用WHERE条件来过滤不匹配的记录。这是在存储引擎层完成的。
* 使用索引覆盖扫描（在Extra列中出现了Using index）来返回记录，直接从索引中过滤不需要的记录并返回命中的结果。这是在MySQL服务器层来完成的，但无需再返回查询记录。
* 从数据表中返回数据，然后过滤不满足条件的记录（在Extra列中出现Using Where）。这在M月SQL服务器层完成，MySQL需要先从数据表读出记录然后过滤。

#### 重构查询
- - -
1、切分查询

例如删除旧数据，定期地清除大量数据时，如果用一个大的语句一次性完成的话，则可能需要一次锁住很多数据、占满整个事务日志、耗尽系统资源、阻塞很多小的但重要的查询。将一个大的DELETE语句切分成多个较小的查询可以尽可能小地影响MySQL性能。

2、分解关联查询

将一个关联查询分解为多个小的查询有几个优点：

* 让缓存的效率更高。一是应用的缓存可以缓存部分小的查询，从而不需要每次都执行大查询；二是MySQL的查询缓存，如果关联中的某个表发生了变化，那么就无法使用查询缓存了，而拆分后，如果某个表发生改变，那么其他表的查询缓存可能仍然有效。
* 将查询分解后，执行单个查询可以减少锁的竞争。
* 在应用层做关联，可以更容易对数据库进行拆分，更容易做到高性能和可扩展。

3、查询语句优化

**IN子查询优化：**

![](/assets/img/201403200102.png)

UNION查询：

`(SELECT first_name, last_name FROM sakila.actor ORDER BY first_name) 
UNION ALL (SELECT first_name, last_name FROM sakila.actor ORDER BY last_name) LIMIT 20`

重写为：

`(SELECT first_name, last_name FROM sakila.actor ORDER BY first_name LIMIT 20) 
UNION ALL (SELECT first_name, last_name FROM sakila.actor ORDER BY last_name LIMIT 20) LIMIT 20`

可减少中间表的记录数。

**索引合并优化：**

考虑建立联合索引。

**最大值和最小值优化：**

`SELECT MIN(actor_id) FROM actor WHERE first_name = 'kun';`

重写为：

`SELECT actor_id FROM actor USE INDEX(PRIMARY) WHERE first_name = 'kun' LIMIT 1;`

利用了InnoDB的聚簇索引。

**优化LIMIT分页：**

`SELECT film_id, description FROM film ORDER BY title LIMIT 50, 5;`

“延迟关联”重写为：

`SELECT film_id, description FROM film INNER JOIN ( SELECT film_id FROM film ORDER BY title LIMIT 50, 5) AS lim USING(film_id);`

另外一种更好的方法是记录上次查询到的位移量，下次查询另一页时，从该位移量开始计算。

**查询优化器的提示**

官方手册，慢慢看吧：[Chapter 8 Optimization](http://dev.mysql.com/doc/refman/5.7/en/optimization.html)

突然感觉看MySQL官方手册才是硬道理。