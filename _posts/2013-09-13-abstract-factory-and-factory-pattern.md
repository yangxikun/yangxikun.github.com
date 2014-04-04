---
layout: post
title: "工厂模式与抽象工厂模式"
description: ""
category: PHP
tags: [PHP设计模式]
---
{% include JB/setup %}
*学习自：维基百科[抽象工厂](http://zh.wikipedia.org/wiki/%E6%8A%BD%E8%B1%A1%E5%B7%A5%E5%8E%82)、[工厂](http://zh.wikipedia.org/wiki/%E5%B7%A5%E5%8E%82%E6%96%B9%E6%B3%95)和博文[抽象工厂模式-与-工厂方法模式区别](http://blog.csdn.net/wangwenhui11/article/details/3955125)*

**类别：创造型模式**

#### 工厂模式
- - -
内容：用于创造一类产品。

特点：

1. 一个抽象产品类，可以派生出多个具体产品类。
2. 一个抽象工厂类，可以派生出多个具体工厂类。
3. 每个具体工厂类只能创建一个具体产品类的实例。

UML图：
![factory](/assets/img/201309130101.jpg)

示例代码：
{% highlight php linenos %}
<?php
interface Button{}
class WinButton implements Button{}
class LinuxButton implements Button{}
interface ButtonFactory
{
    public function createButton();
}
class WinButtonFactory implements ButtonFactory
{
    public function createButton()
    {
        return new WinButton();
    }
}
class LinuxButtonFactory implements ButtonFactory
{
    public function createButton()
    {
        return new LinuxButton();
    }
}
?>
{% endhighlight %}

#### 抽象工厂模式
- - -
内容：用于创建不同主题的产品。

特点：

1. 多个抽象产品类，每个抽象产品类可以派生出多个具体产品类。
2. 一个抽象工厂类，可以派生出多个具体工厂类。
3. 每个具体工厂类代表一个主题，可以创建多个具体产品类的实例。

UML图：
![abstract factory](/assets/img/201309130102.jpg)

示例代码：
{% highlight php linenos %}
<?php
abstract class AbstractFactory {
    abstract public function createButton();
    abstract public function createBorder();
}
 
class LinuxFactory extends AbstractFactory{
    public function createButton()
    {
 
        return new LinuxButton();
    }
    public function createBorder()
    {
        return new LinuxBorder();
    }
}
class WinFactory extends AbstractFactory{
    public function createButton()
    {
        return new WinButton();
    }
    public function createBorder()
    {
        return new WinBorder();
    }
}
abstract class Button{}
abstract class Border{}
 
class LinuxButton extends Button{
    function __construct()
    {
        echo 'LinuxButton is created' . "\n";
    }
}
class LinuxBorder extends Border{
    function __construct()
    {
        echo 'LinuxBorder is created' . "\n";
    }
}
 
 
class WinButton extends Button{
    function __construct()
    {
        echo 'WinButton is created' . "\n";
    }
}
class WinBorder extends Border{
    function __construct()
    {
        echo 'WinBorder is created' . "\n";
    }
}

$type = 'Linux'; //value by user.
if(!in_array($type, array('Win','Linux')))
    die('Type Error');
$factoryClass = $type.'Factory';
$factory=new $factoryClass;
$factory->createButton();
$factory->createBorder();
?>
{% endhighlight %}