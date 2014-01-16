---
layout: post
title: "Pear编码标准 1-9"
description: ""
category: PHP
tags: [PHP编码规范]
---
{% include JB/setup %}
*翻译自[Coding Standards](http://pear.php.net/manual/en/standards.php)的1-9小节*

#### 缩进和行的长度
- - -
使用4个空格的缩进，且不含制表符。这有助于避免差别、补丁、SVN历史和注释带来的问题。为了增强代码的可读性，特别推荐每一行大约占75-85个字符长。 Paul M. Jones有关于这个限制的一些[想法](http://paul-m-jones.com/archives/category/programming/php)。

<!--more-->
#### 控制结构
- - -
包括if、for、while、switch等。这里有一个例子：

{% highlight php linenos %}
<?php
if ((condition1) || (condition2)) {
    action1;
} elseif ((condition3) && (condition4)) {
    action2;
} else {
    defaultaction;
}
?>
{% endhighlight %}

控制语句在关键字和左括号之间应该有一个空格，以区分函数调用。
强烈建议你在每一个控制语句中都使用大花括号，因为不但可以增加可读性，还能防止在添加新一行代码时导致逻辑错误。

对于switch语句：
{% highlight php linenos %}
<?php
switch (condition) {
case 1:
    action1;
    break;

case 2:
    action2;
    break;

default:
    defaultaction;
    break;
}
?>
{% endhighlight %}

#### 将过长的if语句分割几行
- - -
过长的if语句应该被分割为几行，当超过一行的长度限制时（75-85）。在新的行上，应该以4个空格缩进，并且逻辑操作符（&&、||等）应该位于行开头。右括号和左花括号独占一行。

保持操作符位于行开头有两个好处：它能够让你在注释掉条件判断中的某一行时保持正确的语法。另外判断逻辑显得十分清晰。
{% highlight php linenos %}
<?php

if (($condition1
    || $condition2)
    && $condition3
    && $condition4
) {
    //code here
}
?>
{% endhighlight %}
第一个条件也许可以这样对齐：
{% highlight php linenos %}
<?php

if (   $condition1
    || $condition2
    || $condition3
) {
    //code here
}
?>
{% endhighlight %}
最好的情况下，当然是不需要进行分割。当if语句真的很长时，一个有效的方法就是简化它。在这个例子中，你需要多添加几个变量来表示条件判断。这有利于对条件进行命名和分割，如下：
{% highlight php linenos %}
<?php

$is_foo = ($condition1 || $condition2);
$is_bar = ($condition3 && $condtion4);
if ($is_foo && $is_bar) {
    // ....
}
?>
{% endhighlight %}

#### 三元运算符
- - -
用于if语句的规则同时也适用于三元运算符：它也可能被分为几行，保持问号和冒号在前面。
{% highlight php linenos %}
<?php

$a = $condition1 && $condition2
    ? $foo : $bar;

$b = $condition3 && $condition4
    ? $foo_man_this_is_too_long_what_should_i_do
    : $bar;
?>
{% endhighlight %}

#### 函数调用
- - -
在调用函数时，函数名和左括号，第一个参数之间不能有空格，空格应该在每个参数之间，最后一个参数和右括号和分号也不能有空格。例如：
{% highlight php linenos %}
<?php
$var = foo($bar, $baz, $quux);
?>
{% endhighlight %}
正如上例所示，在等号两边都有空格。如果一个语句块有多个赋值语句，应该将等号对齐：
{% highlight php linenos %}
<?php
$short         = foo($bar);
$long_variable = foo($baz);
?>
{% endhighlight %}
为了更佳的可读性，连续几行调用同一个函数/方法时，参数也应该对齐：
{% highlight php linenos %}
<?php
$this->callSomeFunction('param1',     'second',        true);
$this->callSomeFunction('parameter2', 'third',         false);
$this->callSomeFunction('3',          'verrrrrrylong', true);
?>
{% endhighlight %}

#### 将函数调用分隔为几行
- - -
当一个函数调用过长时，应该将其分隔为几行写：
{% highlight php linenos %}
<?php
$this->someObject->subObject->callThisFunctionWithALongName(
    $parameterOne, $parameterTwo,
    $aVeryLongParameterThree
);
?>
{% endhighlight %}
多个参数在同一行是允许的。参数需要相对于函数调用行开头缩进4个空格。左括号位于函数调用行尾，右括号和分号独占一行。

这些规则也同样适用与嵌套函数调用和数组：
{% highlight php linenos %}
<?php
$this->someObject->subObject->callThisFunctionWithALongName(
    $this->someOtherFunc(
        $this->someEvenOtherFunc(
            'Help me!',
            array(
                'foo'  => 'bar',
                'spam' => 'eggs',
            ),
            23
        ),
        $this->someEvenOtherFunc()
    ),
    $this->wowowowowow(12)
);
?>
{% endhighlight %}

当有串连的函数调用时，也可能要分隔为几行。每一个后续行都相对于函数调用行开头缩进4个空格，并且以`->`开头。
{% highlight php linenos %}
<?php

$someObject->someFunction("some", "parameter")
    ->someOtherFunc(23, 42)
    ->andAThirdFunction();
?>
{% endhighlight %}

#### 赋值语句的对齐
- - -
为了更佳的可读性，相邻的赋值语句应该对齐：
{% highlight php linenos %}
<?php
$short  = foo($bar);
$longer = foo($baz);
?>
{% endhighlight %}
但是这个规则可以被打破，当第二个变量的名字比第一个变量的名字的长度超过/短于8个字符的时候：
{% highlight php linenos %}
<?php
$short = foo($bar);
$thisVariableNameIsVeeeeeeeeeeryLong = foo($baz);
?>
{% endhighlight %}

#### 分隔过长的赋值语句
- - -
等号应位于后续行的开头，并且相对左值缩进4个空格：
{% highlight php linenos %}
<?php
$GLOBALS['TSFE']->additionalHeaderData[$this->strApplicationName]
    = $this->xajax->getJavascript(t3lib_extMgm::siteRelPath('nr_xajax'));
?>
{% endhighlight %}

#### 类的定义
- - -
类定义的左花括号要独占一行：
{% highlight php linenos %}
<?php
class Foo_Bar
{

    //... code goes here

}
?>
{% endhighlight %}

#### 函数定义
- - -
函数定义遵循“K&R风格”：
{% highlight php linenos %}
<?php
function fooFunction($arg1, $arg2 = '')
{
    if (condition) {
        statement;
    }
    return $val;
}
?>
{% endhighlight %}
有默认值的参数应该位于参数列表的后面。尽量让函数返回一个有意义的值。下面是一个稍长的例子：
{% highlight php linenos %}
function connect(&$dsn, $persistent = false)
{
    if (is_array($dsn)) {
        $dsninfo = &$dsn;
    } else {
        $dsninfo = DB::parseDSN($dsn);
    }

    if (!$dsninfo || !$dsninfo['phptype']) {
        return $this->raiseError();
    }

    return true;
}
{% endhighlight %}

#### 将过长的函数定义分隔为多行
- - -
函数可能有过多的参数，导致一行超过80个字符。分隔的时候，后续行应相对于`function`关键字缩进4个空格，多个变量允许位于同一行.右括号和左花括号独占一行，并且与`function`关键字保持相同的缩进。
{% highlight php linenos %}
<?php
function someFunctionWithAVeryLongName($firstParameter = 'something', $secondParameter = 'booooo',
    $third = null, $fourthParameter = false, $fifthParameter = 123.12,
    $sixthParam = true
) {
    //....
?>
{% endhighlight %}

#### 数组
- - -
数组赋值也要对齐。当分隔一个数组定义为多个行时，最后的值也应该尾随逗号。
{% highlight php linenos %}
<?php
$some_array = array(
    'foo'  => 'bar',
    'spam' => 'ham',
);
?>
{% endhighlight %}

#### 注释
- - -
建议使用C风格的注释`/* */`和C++风格的注释`//`，不建议使用`#`风格的注释。

#### 包含代码文件
- - -
建议使用`require\_once`和`include\_once`，而且不要括号，因为它们是语法结构，不是函数。

#### PHP代码标签
- - -
必须使用`<?php ?>`。