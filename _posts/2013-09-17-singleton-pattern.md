---
layout: post
title: "单例模式"
description: ""
category: PHP
tags: [PHP设计模式]
---
{% include JB/setup %}

*学习自《Guide to PHP Design Patterns》*

###什么是单例模式?

>一个类在整个PHP程序运行过程中,只需要被实例化一次,这些类通常是提供一些功能,且在程序中需要多次调用,最经典的莫过于数据库连接类.

<!--more-->
###单例模式示例

{% highlight php linenos %}
<?php
/**
 * 数据库连接类
 */
class DbConn {
    /**
     * @static class 当前类的实例
     */
    static private $_instance = NULL;

    /**
     * 私有构造器,使其无法通过new实例化类
     * 
     * @return NULL
     */
    private function __construct(){
    }

    /**
     * 返回当前类的实例
     * 
     * @static
     * @return class 当前类的实例
     */
    static public function getInstance(){
      if ( !self::$_instance ) {
        DbConn::$_instance = new DbConn;
      }
      return self::$_instance;
    }
}
?>
{% endhighlight %}