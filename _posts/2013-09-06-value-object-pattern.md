---
layout: post
title: "值对象模式"
description: ""
category: PHP
tags: [PHP设计模式]
---
{% include JB/setup %}


*学习自《Guide to PHP Design Patterns》*

###什么是值对象模式？

>像PHP的整型那样运作：如果你把同一个对象资源赋值给两个不同的变量，然后改变其中的一个变量，另一个变量仍然不受影响。事实上，这就是Value Object模式的目标所在。*\(这个有点像PHP变量的引用计数，写时拷贝\)*

###来看一个有bug的程序例子

{% highlight php linenos %}
<?php
/**
 * 货币类
 */

class BadDollar {

  /**
   * 货币数量
   *
   * @var Float
   */
  protected $amount;

  /**
   * 构造函数
   *
   * @return Null
   */
  public function __construct( $amount = 0 ) {
    $this->amount = (float)$amount;
  }

  /**
   * 返回$amount的值 
   *
   * @return Float
   */
  public function getAmount() {
    return $this->amount;
  }

  /**
   * 将$amount的值增加
   *
   * @return Null
   */
  public function add( $dollar ) {
    $this->amount += $dollar->getAmount();
  }

}

/**
 * 工作类
 */
class Work {

  /**
   * 工资
   *
   * @var Float
   */
  protected $salary;

  /**
   * 构造函数
   *
   * @return Null
   */
  public function __construct() {
    $this->salary = new BadDollar( 200 );
  }

  /**
   * 支付薪水
   *
   * @return class BadDollar
   */
  public function payDay() {
    return $this->salary;
  }

}

/**
 * 职工类
 */
class Person {

  /**
   * 钱包
   *
   * @var class BadDollar
   */
  public $wallet;
}

/**
 * 测试类
 */
class StackTest extends PHPUnit_Framework_TestCase{
  /**
   * 测试BadDollar类的正确性
   */
    public function testBadDollarWorking(){
        $job = new Work;
        $p1 = new Person;
        $p2 = new Person;
        $p1->wallet = $job->payDay();
        $this->assertEquals( 200, $p1->wallet->getAmount() );
        $p2->wallet = $job->payDay();
        $this->assertEquals( 200, $p2->wallet->getAmount() );
        $p1->wallet->add($job->payDay());
        $this->assertEquals( 400, $p1->wallet->getAmount() );
        //this is bad — actually 400
        $this->assertEquals( 200, $p2->wallet->getAmount() );
        //this is really bad — actually 400
        $this->assertEquals( 200, $job->payDay()->getAmount() );
    }
}
?>
{% endhighlight %}

>从上面的代码和测试中可以看出,$p1和$p2拥有相同的BadDollar实例.为什么会这样呢?因为PHP5中,对象实例赋值给变量是按引用赋值的,且修改了对象实例的内容时,并不会像普通类型变量一样发生"写时复制,"值对象模式就是为了实现对象的"写时复制".

###解决方案

>修改BadDollar类的add方法,如下

{% highlight php linenos %}
<?php
public function add( $dollar ) {
    return new BadDollar( $this->amount + $dollar->getAmount() );
}
?>
{% endhighlight %}

>修改测试用例中的`$p1->wallet->add($job->payDay());`为`$p1->wallet = $p1->wallet->add($job->payDay());`.