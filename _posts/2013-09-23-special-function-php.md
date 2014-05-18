---
layout: post
title: "PHP中比较少用但很有用的几个函数"
description: ""
category: PHP
tags: [PHP练手]
---
{% include JB/setup %}

*转载自[8个开发必备的PHP 功能](http://www.admin10000.com/document/2610.html)*

###函数接收任意数量的参数

{% highlight php linenos %}
<?php
function funtest() {
    $args = func_get_args();
    foreach ( $args as $k => $v ) {
        echo 'arg'.($k+1).': '.$v.'<br />';
    }
}
funtest();
funtest( 'hello' );
funtest( 'hello', 'world', 'next' );
?>
{% endhighlight %}

<!--more-->

###使用glob\(\)查找文件

{% highlight php linenos %}
<?php
//取得当前目录下后缀名为.php的文件
$files = glob( '*.php' );
echo '<pre>';
print_r( $files );

//当前目录下后缀名为.php和.html的文件
$files = glob( '*.{php,html}', GLOB_BRACE );
print_r( $files );

//还可以添加路径
$files = glob( './testdrive/i*.php' );
print_r( $files );

//如果你想得到绝对路径,你可以调用realpath()函数
$files = glob( './testdrive/i*.php' );
$files = array_map( 'realpath', $files );
print_r( $files );
?>
{% endhighlight %}

###生成唯一的id 

>很多朋友都利用md5()来生成唯一的编号，但是md5()有几个缺点：1.无序，导致数据库中排序性能下降。2.太长，需要更多的存储空间。其实PHP中自带一个函数来生成唯一的id，这个函数就是uniqid() 。下面是用法：

{% highlight php linenos %}
<?php
// generate unique string
echo uniqid().'<br />';

// generate another unique string
echo uniqid().'<br />';

//该算法是根据CPU时间戳来生成的，所以在相近的时间段内，id前几位是一样的，这也方便id的排序，如果你想更好的避免重复，可以在id前加上前缀，如：

// 前缀
echo uniqid('foo_').'<br />';

// 有更多的熵
echo uniqid('',true).'<br />';

// 都有
echo uniqid('bar_',true).'<br />';
?>
{% endhighlight %}

###字符串压缩

>当我们说到压缩，我们可能会想到文件压缩，其实，字符串也是可以压缩的。PHP提供了gzcompress\(\) 和 gzuncompress\(\) 函数：

{% highlight php linenos %}
<meta charset="utf-8">
<?php
$string =
'Lorem ipsum dolor sit amet, consectetur
adipiscing elit. Nunc ut elit id mi ultricies
adipiscing. Nulla facilisi. Praesent pulvinar,
sapien vel feugiat vestibulum, nulla dui pretium orci,
non ultricies elit lacus quis ante. Lorem ipsum dolor
sit amet, consectetur adipiscing elit. Aliquam
pretium ullamcorper urna quis iaculis. Etiam ac massa
sed turpis tempor luctus. Curabitur sed nibh eu elit
mollis congue. Praesent ipsum diam, consectetur vitae
ornare a, aliquam a nunc. In id magna pellentesque
tellus posuere adipiscing. Sed non mi metus, at lacinia
augue. Sed magna nisi, ornare in mollis in, mollis
sed nunc. Etiam at justo in leo congue mollis.
Nullam in neque eget metus hendrerit scelerisque
eu non enim. Ut malesuada lacus eu nulla bibendum
id euismod urna sodales. ';
$compressed = gzcompress( $string );
echo 'Original size: '. strlen( $string ).'<br />';

echo 'Compressed size: '. strlen( $compressed ).'<br />';

echo 'Compressed string:<br />'.$compressed;
// 解压缩
$original = gzuncompress( $compressed );
echo '<br /><br />Uncompressed string:<br />'.$original;
?>
{% endhighlight %}

>几乎有50% 压缩比率。同时，你还可以使用gzencode\(\)和gzdecode\(\)函数来压缩，只不过其用了不同的压缩算法。