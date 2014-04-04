---
layout: post
title: "带着操作系统的知识编程"
description: ""
category: PHP
tags: [PHP应用]
---
{% include JB/setup %}

###标题解释

>在coding时,有时遇到的问题可以用操作系统的知识来解决.

###问题描述

>在一个PHP代码文件里有两段主要代码:其一是提供A数据\(这部分代码给个昵称为ACode\),其二是提供B数据\(这部分代码给个昵称为BCode\).其中BCode需要A数据中的部分数据\(给个昵称为need\)才能计算出B数据.

>那么通常的做法就是先执行ACode得到A数据后,BCode再执行不就行了?可如果出现下面这种情况:

>假设ACode执行到得到need的时间为t1,执行得到A数据的时间为t2,且t1小于t2很多,那么BCode就要等待很久才执行.也许你会想:那就把BCode放在ACode得到need数据之后执行,但这样的话,ACode剩下的那部分代码就要等到BCode执行完后再执行.

>现在需要的实现是:ACode在得到need的时候,BCode就能很快开始执行,而ACode剩下的代码可以和BCode并行执行.

<!--more-->

###如何实现?

>模拟操作系统进程同步的实现,这里将ACode比喻为ACode进程,BCode比喻为BCode进程.

>从问题描述可以看出ACode进程和BCode进程之间存在同步关系,即BCode需要ACode计算出的need数据,且他们之间的制约关系为:直接相互制约\(源于进程间的合作\).

>为了实现ACode进程和BCode进程能在一定程度上并行,需要提供一个输入缓冲区.当ACode进程计算得到need数据后就放入缓冲区中,BCode进程"监听"缓冲区,一旦有数据存入,便取出并计算出B数据.

###PHP代码要如何设计?

>首先将ACode和BCode分为两个php代码文件,这里我模拟的是两个ajax并发请求,其中A请求从服务器获取A数据,B请求从服务器获取B数据,ACode和BCode之间的"缓冲区"使用memcached实现.

>前台代码:

{% highlight html linenos %}
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
<html>
    <head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
    <title>Wp Migration</title>
        <script src="http://code.jquery.com/jquery-1.9.1.js"></script>
            <script>
                $(function(){
                    $('#myForm').on('click',function(e){
                         $.ajax({
                            url:'ACode.php',
                            data:$(this).serialize(),
                            type:'GET',
                            success:function(result){
                                $('#result').append(result);
                            }
                        });
                         $.ajax({
                            url:'BCode.php',
                            data:$(this).serialize(),
                            type:'GET',
                            success:function(result){
                                $('#result').append(result);
                            }
                        });
                        return false;
                    });
                });
        </script>
    </head>
    <body>
        <button id="myForm">click</button>
        <div id="result"></div>
    </body>
</html>
{% endhighlight %}

>ACode代码:

{% highlight php linenos %}
<?php 
    $memcached = new memcached( 'fetch' );
    $memcached->addServer( '127.0.0.1', 11211 );
    $key = '123456'; //ACode和BCode通信的标识
    $sellerId = 456789;//need数据
    if ( $memcached->add( $key, $sellerId , 100 ) ){
        echo json_encode( array( 'addOk' ) );
    }else{
        echo json_encode( array( 'hasAdd' ) );
    }
?>
{% endhighlight %}

>BCode代码:

{% highlight php linenos %}
<?php
    $memcached = new memcached( 'fetch' );
    $memcached->addServer( '127.0.0.1', 11211 );
    $key = '123456'; //ACode和BCode通信的标识
    while ( ($value = $memcached->get( $key )) == FALSE) {//"监听"直到ACode将need数据存入"缓冲区"
        //这里可以用sleep()函数设置多久检测一次
    }
    echo json_encode( array($value) );
?>
{% endhighlight %}