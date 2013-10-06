---
layout: post
title: "正则表达式小结"
description: ""
category: PHP
tags: [PHP应用]
---
{% include JB/setup %}
<style>
    tr,td {
        border: 1px solid black
    }
</style>
###分隔符

>当使用 PCRE 函数的时候，模式需要由分隔符闭合包裹。分隔符可以使任意非字母数字、非反斜线、非空白字符。经常使用的分隔符是正斜线(/)、hash符号(#) 以及取反符号(~)。

###元字符

>元字符是正则表达式中具有特殊意义的专用字符,用来规定其前导字符在目标对象中的出现模式.

>共有两种不同的元字符：一种是可以在模式中方括号外任何地方使用的，另外一种 是需要在方括号内使用的。(元字符列表)[http://www.php.net/manual/zh/regexp.reference.meta.php]

###字符组

>使用中括号的字符集称为字符组.例如`[c[aou?*)]]t`,注意字符组里的元字符可以不转义.在字符组中要使用`\`时,在其前面要多加`\\`两个斜线.

###分支

>在圆括号中使用|进行分支.例如:`(c|h|to)at`,可以匹配`cat`,`hat`,`toat`.

###分组

>通过圆括号分隔界定，并且它们可以嵌套.常用分组语法:

<table cellspacing="0">
<tbody>
<tr>
<td>分类</td>
<td>代码/语法</td>
<td>说明</td>
</tr>
<tr>
<td rowspan="3">捕获</td>
<td>(exp)</td>
<td>匹配exp,并捕获文本到自动命名的组里</td>
</tr>
<tr>
<td>(?&lt;name&gt;exp)</td>
<td>匹配exp,并捕获文本到名称为name的组里，也可以写成(?’name’exp)</td>
</tr>
<tr>
<td>(?:exp)</td>
<td>匹配exp,不捕获匹配的文本，也不给此分组分配组号</td>
</tr>
<tr>
<td rowspan="4">断言</td>
<td>(?=exp)</td>
<td>匹配exp前面的位置</td>
</tr>
<tr>
<td>(?&lt;=exp)</td>
<td>匹配exp后面的位置</td>
</tr>
<tr>
<td>(?!exp)</td>
<td>匹配后面跟的不是exp的位置</td>
</tr>
<tr>
<td>(?&lt;!exp)</td>
<td>匹配前面不是exp的位置</td>
</tr>
<tr>
<td rowspan="1">注释</td>
<td>(?#comment)</td>
<td>这种类型的分组不对正则表达式的处理产生任何影响，用于提供注释让人阅读</td>
</tr>
</tbody>
</table>

>可以看看这篇博文:[正则表达式 – 分组语法（捕获/断言/注释）](http://www.anrip.com/post/1126)

###反向引用

>注意它不仅可以用在pattern中,还可以用在replacement中,例如:

{% highlight php linenos %}
<?php
$str = '\*?123';
$str = preg_replace('~[\\\*?]+(\d+)~', '\1', $str);
var_dump($str);
?>
{% endhighlight %}

###贪婪/懒惰匹配模式

>在重复量词之后加个?就形成了懒惰模式,例如:`a.*b`匹配`aabab`整个字符串,而`a.*?b`匹配`aabab`的`aab`和`ab`两组字符.

###常用的模式

>1.忽略大小写模式\(i\)
>2.多行模式\(m\)
>3.点号通配模式\(s\),即点号可以匹配换行符
>4.懒惰模式\(U\),相当于重复量词之后加了个?
>5.支持UTF-8转义表达\(u\)

{% highlight php linenos %}
<?php
$str = '你好世界';
preg_match('~[\x{4e00}-\x{9fa5}]+~u', $str, $matches);
var_dump($matches);
?>
{% endhighlight %}

###正则表达式的效率与优化

>1.使用字符组会比分支条件快一点<br />
>2.尽量使用与匹配字符相同类型的元字符<br />
>3.匹配字符数目确定的就不用任意多量词<br />
>4.尽量使用字符串处理函数代替正则<br />
>5.合理使用括号,每使用一个普通括号,而不是非捕获型括号`(?:)`,就会保留一部分内存等着再次被访问<br />
>6.使用PHP原生函数[Filter系列函数](http://www.php.net/manual/zh/ref.filter.php)和[ctype函数](http://www.php.net/manual/zh/ref.ctype.php)