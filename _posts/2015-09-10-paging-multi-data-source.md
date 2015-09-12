---
layout: post
title: "多表数据分页方案"
description: ""
category: 
tags: []
---
{% include JB/setup %}

通常情况下，只需要对单表的数据进行分页：

`SELECT …… LIMIT ($CURRENT_PAGE - 1) * $PAGE_SIZE, $PAGE_SIZE ORDER BY ……`

然而，在比较复杂的业务场景下，数据来自多张表（并非水平分割的表），这时就要考虑多张表的情况下，数据聚合到一起后如何分页了。

这里给出一个参考方案：

假设数据来自A、B表，分页规则按照记录的创建时间排序，再定义一个分页的状态变量time（记录时间）：

* 当获取第1页数据时：`SELECT …… FROM A WHERE …… LIMIT 0, $PAGE_SIZE ORDER BY ……` 和 `SELECT …… FROM B WHERE …… LIMIT 0, $PAGE_SIZE ORDER BY ……`，将这两份数据在代码层面合并到一起后取出前`$PAGE_SIZE`条记录items，并且记下第`$PAGE_SIZE`条记录的创建时间为time，再记下items中最后一条A记录的id为aid和最后一个B记录的id为bid；
* 当获取第2页数据时：`SELECT …… FROM A WHERE …… AND created <= time AND id != aid LIMIT $PAGE_SIZE * 1, $PAGE_SIZE ORDER BY ……`和`SELECT …… FROM B WHERE …… AND created <= time AND id != bid LIMIT $PAGE_SIZE * 1, $PAGE_SIZE ORDER BY ……`，将这两份数据在代码层面合并到一起后取出前`$PAGE_SIZE`条记录items，同时更新aid和bid。
* 之后数据的获取以此类推。

之所以记录aid和bid是考虑到同一张表里可能有created相同的记录，并且处于分页的边缘。
