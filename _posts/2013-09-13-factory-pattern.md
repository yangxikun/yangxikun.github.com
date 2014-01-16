---
layout: post
title: "工厂模式"
description: ""
category: PHP
tags: [PHP设计模式]
---
{% include JB/setup %}

*学习自《Guide to PHP Design Patterns》*

>在面向对象编程中,我们实例化一个类最普遍使用的就是new操作符.但在某些情况下,new操作符也许不太好用.

>例如,当你需要对一个变量进行赋值一个实例化对象时,例如有productA和productB两个类,变量需要根据条件赋值相应的类实例,那么也许代码会是这样子:

<!--more-->
{% highlight php linenos %}
<?php
if( condition ){
    $var = new productA();
}else{
    $var = new productB();
}
?>
{% endhighlight %}

>如果使用了工厂模式的话,代码应该如下这样子:

{% highlight php linenos %}
<?php
$var = Product->create( $condition );
?.
{% endhighlight %}

>显然,代码简洁了许多,还有其他一些好处,比如现在productA不需要了,那么在代码中,你是否要得删除所有new productA的语句?但如果在工厂模式下,你只需要修改工厂的create方法.

>一般会在以下几种情况考虑使用工厂方法:

>>1.当一个类的初始化需要经过许多复杂的计算或依赖于其他类时;

>>2.当需要动态地为变量实例化某个类的子类时;

>工厂模式所带来的好处:增强系统的可扩展性和编码时尽量少的修改量.

###简单的工厂示例:

{% highlight php linenos %}
<?php
/**
 * 产品基类
 * 
 * @abstract
 */
abstract class Product {

  /**
   * 获取产品的名字
   * 
   * @abstract
   */
  abstract function getName();
}

/**
 * A产品
 */
class ProductA extends Product {

  /**
   * 获取产品的名字
   * 
   * @return string
   */
  public function getName() {
    echo 'Product A';
  }

}

/**
 * B产品
 */
class ProductB extends Product {

  /**
   * 获取产品的名字
   * 
   * @return string
   */
  public function getName() {
    echo 'Product B';
  }

}

/**
 * 工厂类,用于生产产品
 */
class ProductFactory {

  /**
   * 创建product子类
   * 
   * @param mixed $condition 具体条件
   * @return Product
   */
  public function create( $condition ) {
    if ( $condition ) {
      return new ProductA();
    }else{
      return new ProductB();
    }
  }
  
}
?>
{% endhighlight %}

>复杂的工厂模式可能会定义接口类工厂或抽象类工厂,以及复杂的创建条件.