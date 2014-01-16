---
layout: post
title: "递归遍历目录"
description: ""
category: PHP
tags: [PHP练手]
---
{% include JB/setup %}

###方法一

{% highlight php linenos %}
<meta charset="utf-8">
<?php 
function showDir( $dir )
{
    $r = array();
    foreach ( scandir( $dir ) as $key => $value ) {// scandir()列出指定路径中的文件和目录
        if ( $value === '.' || $value === '..'  ) {
            continue;
        }
        if ( is_dir( $dir.'/'.$value ) ) {
            $r[$dir.'/'.$value] = showDir( $dir.'/'.$value );
        }
    }
    return $r;
}
var_dump( showDir( '.' ) );
?>
{% endhighlight %}

<!--more-->
###方法二

{% highlight php linenos %}
<meta charset="utf-8">
<?php
function showDir( $dir )
{
    $r              = array();
    $handledir = opendir( $dir ) ;
    while ( false !== $dirName = readdir( $handledir ) ) {//readdir()返回目录中下一个文件的文件名。文件名以在文件系统中的排序返回。 
        if ( $dirName==='.' || $dirName=='..' ) {
            continue;
        }
        if ( is_dir( $dir.'/'.$dirName ) ) {
            $r[$dir.'/'.$dirName] = showDir( $dir.'/'.$dirName );
        }
    }
    return $r;
}
var_dump( showDir( '.' ) );
?>
{% endhighlight %}