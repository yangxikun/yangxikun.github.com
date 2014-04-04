---
layout: post
title: "PHP变量引用计数，写时复制总结"
description: ""
category: PHP
tags: [PHP扩展]
---
{% include JB/setup %}
*学习自[php-variables](http://derickrethans.nl/talks/phparch-php-variables-article.pdf)*

在PHP的变量存储结构ZVAL中，有两个成员：`refcount__gc`和`is_ref_gc`用于实现引用计数和写时复制，这样既能节省内存，又能减少计算量。

当执行如下代码时：
{% highlight php linenos %}
<?php
$a = "this is";
$b = "variable";
$c = 42;
?>
{% endhighlight %}
三个变量各自拥有一个ZVAL，如下图所示：<br>
![demo](/assets/img/201401280101.png)

<!--more-->
当执行如下代码时，将会发生写时复制：
{% highlight php linenos %}
<?php
$a = "this is";
$b = $a;
$c = $a;
$c = 42;
unset($b);
unset($c);
?>
{% endhighlight %}
$a、$b、$c先指向同一个ZVAL，在修改$c的值时，就需要为$c复制一个ZVAL了，当执行`unset()`操作时，将会断开变量和ZVAL的”关联“。refcount\_\_gc减1，如果refcount\_\_gc为0，那么PHP的就会将其当作垃圾，在适当的时候回收。如图：<br>
![demo](/assets/img/201401280102.png)

当变量作为参数传递给函数时，想想会怎样：
{% highlight php linenos %}
<?php
function do_something($s)
{
    $s = 'was';
    return $s;
}

$a = 'this is';
$b = do_something($a);
?>
{% endhighlight %}
其实变量作为参数传递时，也可以看作是一个变量赋值过程（`$s = $a`），当这时，refcount\_\_gc却要加2，为什么？看下图：<br>
![demo](/assets/img/201401280103.png)

再来看看使用`&`引用变量的例子：
{% highlight php linenos %}
<?php
$a = "this is";
$b = &$a;
$c = &$a;
$b = 42;
unset($c);
unset($a);
?>
{% endhighlight %}
当使用`&`引用变量时，refcount\_\_gc加1。另外，is\_ref\_\_gc会被**设置**为1，不管有1个还是更多个变量使用`&`引用了同一个ZVAL。看图：<br>
![demo](/assets/img/201401280104.png)

还有两个有趣的例子，`&`操作符也会引起ZVAL复制：
{% highlight php linenos %}
<?php
$a = "this is";
$b = $a;
$c = &$b;
?>
{% endhighlight %}

![demo](/assets/img/201401280105.png)

{% highlight php linenos %}
<?php
$a = "this is";
$b = &$a;
$c = $a;
?>
{% endhighlight %}
![demo](/assets/img/201401280106.png)

那么如果通过引用传递参数，又会如何？
{% highlight php linenos %}
<?php
function do_something(&$s)
{
    $s = 'was';
    return $s;
}
$a = 'this is';
$b = do_something($a);
?>
{% endhighlight %}
从PHP 5.4开始已经不允许使用`foo(&$var)`形式的函数调用了，这将会导致致命错误！see [Passing by Reference](http://cn2.php.net/manual/en/language.references.pass.php)<br>
![demo](/assets/img/201401280107.png)

那么通过引用返回时，又会怎样？
{% highlight php linenos %}
<?php
function &find_node($key, &$node)
{
    $item = & $node[$key];
    return $item;
}

$tree = array(
    1 => 'one',
    2 => 'two',
    3 => 'three',
);
$node = & find_node(3, $tree);
$node = 'drie';
?>
{% endhighlight %}
一张很长的图：<br>
![demo](/assets/img/201401280108.png)
![demo](/assets/img/201401280111.png)
![demo](/assets/img/201401280112.png)

对于在函数内使用global关键字引用的全局变量：
{% highlight php linenos %}
<?php
$var = 'one';

function update_var($val)
{
    global $var;
    unset($var);
    global $var;
    $var = $val;
}

update_var("four");
?>
{% endhighlight %}

![demo](/assets/img/201401280109.png)
![demo](/assets/img/201401280113.png)
![demo](/assets/img/201401280114.png)

最后这个例子是为了说明，在PHP里，有些时候定义函数返回引用是毫无意义的，并不能达到节省内存的目的，所以要慎用函数引用返回，先看代码：
{% highlight php linenos %}
<?php
function &definition()
{
    $def = array('id' => 'name', 'def' => 42);
    return $def;
}

$def &definition();
?>
{% endhighlight %}
如果对引用计数，写时复制有了充分了解，那么应该能看出上面的代码实际执行时和不使用引用返回是一样的效果，并不会多消耗ZVAL。<br>
![demo](/assets/img/201401280110.png)