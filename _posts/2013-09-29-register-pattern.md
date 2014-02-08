---
layout: post
title: "注册表模式"
description: ""
category: PHP
tags: [PHP设计模式]
---
{% include JB/setup %}

*学习自《Guide to PHP Design Patterns》*

#### 内容
- - -
注册表模式用于访问全局可重用的对象,可以容纳多个类的多个实例.这些类的实例一般都是经常被调用到或当做参数传递的.

基本的注册表模式用于保持全局存储,需要以下4种实现方法:

<!--more-->
1. Register::add()添加一个对象到注册表中,可以指定一个名称(一个类的多个实例)或者使用默认的类名(类似单例)
2. Register::get()从注册表中检索一个对象
3. Register::contains()在注册表中检查一个对象是否存在
4. Register::remove()通过对象名在注册表中删除一个对象

示例代码:
{% highlight php linenos %}
<?php 
/**
 * 注册表类
 */
class Register {

    /**
     * @static
     * @var array 存储实例
     */
    static private $_store = array();

    /**
     * 添加一个对象到注册表中,可以指定一个名称(一个类的多个实例)或者使用默认的类名(类似单例)
     * 
     * @param mixed $object 被存储的对象
     * @param string $name 用于检索对象的名字
     * @return mixed 如果重写了实例,那么之前的实例会被返回
     */
    static public function add($object, $name = null) {
        $name   = $name !== null ? : get_class($object);
        $name   = strtolower($name);
        $return = null;
        if (isset(self::$_store[$name])) {
            $return = self::$_store[$name];
        }
        self::$_store[$name] = $object;
        return $return;
    }

    /**
     * 从注册表中检索一个对象
     * 
     * @param string $name 对象名字,{@see self::add()}
     * @return mixed
     * @throws Exception
     */
    static public function get($name){
        if (!self::contains($name)) {
            throw new Exception('对象没有存储在注册表中');
        }
        return self::$_store[$name];
    }

    /**
     * 在注册表中检查一个对象是否存在
     * 
     * @param string $name 对象名称,{@see self::add()}
     * @return bool
     */
    static public function contains($name)
    {
        if (isset(self::$_store[$name])) {
            return true;
        }
        return false;
    }

    /**
     * 通过对象名在注册表中删除一个对象
     * 
     * @param string $name 对象名称,{@see self::add()}
     * @return void
     */
    static public function remove($name)
    {
        if (self::contains($name)) {
            unset(self::$_store[$name]);
        }
    }
}
?>
{% endhighlight %}