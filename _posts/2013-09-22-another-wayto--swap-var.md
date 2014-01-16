---
layout: post
title: "不使用第三个变量交换两个变量的方法"
description: ""
category: PHP
tags: [PHP练手]
---
{% include JB/setup %}

*核心是使用异或操作,将两个变量的值都看成二进制就一目了然了.*

<!--more-->
{% highlight php linenos %}
<meta charset="utf-8">
<?php 
$a = '123abc你好!';
$b = '好啊!456xyz';
echo '$a='.$a.'<br />';
echo '$b='.$b.'<br /><br />';
$a = $a ^ $b;
$b = $b ^ $a;
$a = $a ^ $b;
echo 'swap $a $b<br /><br />';
echo '$a='.$a.'<br />';
echo '$b='.$b.'<br />';
?>
{% endhighlight %}