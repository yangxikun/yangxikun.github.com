---
layout: post
title: "迭代模式"
description: ""
category: PHP
tags: [PHP设计模式]
---
{% include JB/setup %}

*学习自《Guide to PHP Design Patterns》*

>在PHP的SPL中,有iterator这个类库,它的主要用途在于为一组数据集(可以是数组或对象)提供各种遍历的方法.

<!--more-->
###一个简单的迭代

{% highlight php linenos %}
<?php 
class testIterator implements iterator {
    private $_store;//存储数据的集合

    public function __construct (&$arr) {
        $this->_store = $arr;
    }

    public function current () {
        return current($this->_store);
    }

    public function next () {
        next($this->_store);
    }

    public function key() {
        return key($this->_store);
    }

    public function rewind () {
        reset($this->_store);
    }

    public function valid () {//在next()和rewind()调用之后会被自动调用
        return true;
    }
}
$arr         = array(1,2,3,false,4,5);
$arrIterator = new testIterator($arr);
foreach ($arr as $key => $value) {
    echo $key.'-'.$value.'<br />';
}
?>
{% endhighlight %}

>迭代的类库有各种迭代的接口,这也是为了规范开发者编写的代码吧,具体请看PHP手册[iterator](http://cn2.php.net/manual/en/class.iterator.php).