---
layout: post
title: "MySQL EXPLAIN学习"
description: ""
category: MySQL
tags: [MySQL查询优化]
---
{% include JB/setup %}
*参考自：[Mysql Explain 详解](http://www.cnitblog.com/aliyiyi08/archive/2008/09/09/48878.html)和[EXPLAIN Output Format](https://dev.mysql.com/doc/refman/5.7/en/explain-output.html#explain-extra-information)*

EXPLAIN 列输出解析：

#### id
- - -
SELECT 标识，是执行 SELECT 的顺序。该值可以为 NULL ，如果是引用了 union 的结果。在这种情况下，table 列的值应该像`<unionM,N>`，表示被 union 的行的id值为 M 和 N。

<!--more-->
示例图：

![](/assets/img/201403180101.png)

#### select_type
- - -
|值  |意义 |
|----|----|
|SIMPLE|简单的 SELECT，没有使用 UNION 或者 子查询|
|PRIMARY|最外层的 SELECT|
|UNION|在 UNION 中的第二个或者之后的 SELECT|
|DEPENDENT UNION|在 UNION 中的第二个或者之后的 SELECT，依赖于外部查询|
|UNION RESULT|UNION的结果|
|SUBQUERY|在子查询中的第一个 SELECT|
|DEPENDENT SUBQUERY|在子查询中的第一个 SELECT，依赖于外部查询|
|DERIVED|派生表的SELECT（在 FROM 子句中的子查询）|
|MATERIALIZED|物化子查询|
|UNCACHEABLE SUBQUERY|A subquery for which the result cannot be cached and must be re-evaluated for each row of the outer query|
|UNCACHEABLE UNION|The second or later select in a UNION that belongs to an uncacheable subquery (see UNCACHEABLE SUBQUERY)|

示例图：

![](/assets/img/201403180102.png)

为什么会是 Dependent？请查看：[Restrictions on Subqueries](http://dev.mysql.com/doc/refman/5.7/en/subquery-restrictions.html)

![](/assets/img/201403180103.png)

![](/assets/img/201403180104.png)

#### table
- - -
表名，或者以下其中之一：

* `<unionM,N>`：表示被 union 的行的id值为 M 和 N。
* `<derivedN>`：表示从行id为N的查询派生出来的。
* `<subqueryN>`：The row refers to the result of a materialized subquery for the row with an id value of N. 


#### type
- - -
join的类型。

* system：表仅有一行，这是const联接类型的一个特例。

![](/assets/img/201403180105.png)

* const：表最多有一个匹配行，它将在查询开始时被读取。因为仅有一行，在这行的列值可被优化器剩余部分认为是常数。const表很快，因为它们只读取一次！当使用常量与 PRIMARY KEY 或者 UNIQUE 索引比较时，将为 const。

![](/assets/img/201403180106.png)

* eq_ref：当索引为主键索引或者非空唯一索引。

![](/assets/img/201403180107.png)

* ref：当索引不是一个 PRIMARY KEY 或 UNIQUE 索引，而是一个普通索引。

![](/assets/img/201403180108.png)

* fulltext：join 使用了 FULLTEXT 索引。
* ref\_or\_null：该联接类型如同ref，但是添加了额外的查询条件：包含NULL值的行。在解决子查询中经常使用该联接类型的优化。

![](/assets/img/201403180109.png)

* index\_merge：该联接类型表示使用了索引合并优化方法。在这种情况下，key列包含了使用的索引的清单，key_len包含了使用的索引的最长的关键元素。

![](/assets/img/201403180110.png)

* unique\_subquery：该类型替换了下面形式的IN子查询的ref类型：`value IN (SELECT primary_key FROM single_table WHERE some_expr)`。unique\_subquery是一个索引查找函数，可以完全替换子查询，效率更高。

![](/assets/img/201403180111.png)

* index\_subquery：该联接类型类似于unique\_subquery。可以替换IN子查询，但只适合下列形式的子查询中的非唯一索引：`value IN (SELECT key_column FROM single_table WHERE some_expr)`

* range：只检索给定范围的行，使用一个索引来选择行。key列显示使用了哪个索引。key_len包含所使用索引的最长关键元素。在该类型中ref列为NULL。

![](/assets/img/201403180112.png)

* index：该联接类型与ALL相同，除了只有索引树被扫描。出现这种情况有两种方式：
> 1、如果索引的查询是覆盖索引，并且可以被用来满足从表中所要求的所有数据，只有索引树被扫描。在这种情况下，Extra 列的值为 Using index。

> 2、根据索引查找的索引顺序进行全表扫描。这时，Extra 列的值不会出现 Using index。

* ALL：对于每个来自于先前的表的行组合，进行完整的表扫描。如果表是第一个没标记const的表，这通常不好，并且通常在其它情况下很差。通过增加索引来避免这种情况。

![](/assets/img/201403180113.png)

#### possible\_keys
- - -
possible\_keys 列指出MySQL可能使用哪个索引在该表中找到行。注意，该列完全独立于 XPLAIN 输出所示的表的次序。这意味着在 possible\_keys 中的某些键实际上不能按生成的表次序使用。

如果该列是 NULL，则没有相关的索引。在这种情况下，可以通过检查 WHERE 子句看是否它引用某些列或适合索引的列来提高你的查询性能。如果是这样，创造一个适当的索引并且再次用 EXPLAIN 检查查询。

#### key
- - -
key列显示MySQL实际决定使用的键（索引）。如果没有选择索引，键是NULL。要想强制MySQL使用或忽视possible\_keys列中的索引，在查询中使用FORCE INDEX、USE INDEX或者IGNORE INDEX。

#### key_len
- - -
key_len列显示MySQL决定使用的键长度。如果键是NULL，则长度为NULL。
使用的索引的长度。在不损失精确性的情况下，长度越短越好 

#### ref
- - -
显示使用哪个列或常数与索引进行比较，从表中选择行。

#### rows
- - -
显示MySQL认为它执行查询时必须检查的行数。

#### Extra
- - -
该列显示了MySQL执行查询时的额外信息。下面所列的是可能的值，如果你想是查询更快，请注意值为 Using filesort 和 Using temporary 的查询。

*以下并不是全部*

* const row not found：当查询为`SELECT ... FROM tbl_name`时，表为空。
* Distinct：一旦MYSQL找到了匹配的行，就不再搜索了。
* Impossible HAVING：HAVING 条件始终为 false，且没有找到任何行。
* Impossible WHERE：WHERE 条件始终为 false，且没有找到任何行。
* Not exists：MYSQL 优化了 LEFT JOIN，一旦它找到了匹配行就返回。
* Using filesort：看到这个的时候，查询就需要优化了。MYSQL需要进行额外的步骤来发现如何对返回的行排序。它根据连接类型以及存储排序键值和匹配条件的全部行的行指针来排序全部行。
* Using temporary：看到这个的时候，查询就需要优化了。这里，MYSQL需要创建一个临时表来存储结果，这通常发生在进行ORDER BY和GROUP BY的列不同。
* Using index：执行覆盖索引查询时。
* Using where：使用了WHERE从句来限制哪些行将与下一张表匹配或者是返回给用户。
* index_merge：用于通过range扫描搜索行并将结果合成一个。合并会产生并集、交集或者正在进行的扫描的交集的并集。