---
layout: post
title: "urlencode和rawurlencode区别"
description: ""
category: PHP
tags: [PHP应用]
---
{% include JB/setup %}

`urlencode()`：用于编码URL中的参数部分。
`rawurlencode()`：用于编码URL路径部分。
<!--more-->

示例代码：
{% highlight php linenos %}
<?php
$path  = 'test work';
$param = '123 456 789';
$url   = 'http://localhost/'.rawurlencode($path).'.php?id='.urlencode($param);
echo $url."\n";//http://localhost/test%20work.php?id=123+456+789
?>
{% endhighlight %}