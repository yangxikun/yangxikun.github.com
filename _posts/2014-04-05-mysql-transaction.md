---
layout: post
title: "MySQL事务学习"
description: ""
category: MySQL
tags: [事务]
---
{% include JB/setup %}

在MySQL中，通过`START TRANSACTION`或者`BEGIN`来开始一个事务，以`COMMIT`或者`ROLLBACK`来结束一个事务。

MySQL默认的隔离级别为可重复读。

MySQL有一个[变量](https://dev.mysql.com/doc/refman/5.7/en/show-variables.html)`autocommit`，默认值为 ON 。当该值为 ON 时，在每一个会话中，每一次SQL查询都会作为一个单独的事务执行。如果该值为 OFF，那么每一个会话就是一个事务，需要明确执行 commit 或者 rollback 。

<!--more-->

现在假设有这样一个事务：有一个借书系统，每本书在数据库中都有一个 num 字段记录可借的数目。现在书 bookA 的 num 字段为1，有两个读者同时要借这本书。

如果在MySQL中，我们的应用事务按如下执行：

{% highlight php linenos %}
BEGIN;
SELECT num FROM book WHERE name = bookA;//应用中判断 num 是否大于0
UPDATE book SET num = num - 1 WHERE name = bookA;
COMMIT;
{% endhighlight %}

那么最后 bookA 的 num 字段的值将可能为 -1 。因为两个并发的事务可能按如下方式执行：

![](/assets/img/201404050101.png)

原因就在于 MySQL 在读的时候默认加的是共享锁，因为隔离级别默认为“可重复读”。可以通过修改变量`tx_isolation`改变隔离级别，当然也可以像下面那样使用 InnoDB 内置的查询提示：

{% highlight php linenos %}
BEGIN;
SELECT num FROM book WHERE name = bookA FOR UPDATE;//应用中判断 num 是否大于0，将会对行加排他锁
UPDATE book SET num = num - 1 WHERE name = bookA;
COMMIT;
{% endhighlight %}

这样，当有一个事务 T1 先 SELECT 了，那么另外一个事务 T2 将无法 SELECT 直到 T1 完成。