---
layout: post
title: "依赖注入模式"
description: ""
category: PHP
tags: [PHP设计模式]
---
{% include JB/setup %}

###什么是依赖注入模式？

>控制反转（Inversion of Control，英文缩写为IoC）是一个重要的面向对象编程的法则来削减计算机程序的耦合问题。 控制反转还有一个名字叫做依赖注入（Dependency Injection）。简称DI。许多功能都是由两个或是更多的类通过彼此的合作来实现业务逻辑，这使得每个对象都需要，与其合作的对象（也就是它所依赖的对象）的引用。如果这个获取过程要靠自身实现，那么如你所见，这将导致代码高度耦合并且难以测试。

>[What is Dependency Injection?](http://fabien.potencier.org/article/11/what-is-dependency-injection#)这篇文章讲得很好。

>在此对上面那篇文章进行翻译和整理。

>为了克服HTTP协议的无状态性，web应用程序需要一种方法在请求之间存储用户信息。使用cookie来完成确实简单。这里使用PHP内建的session机制。

>假设我们现在有一个用户类

{% highlight php linenos %}
<?php
class User
{
    protected $storage;

    function __construct()
    {
        $this->storage = new SessionStorage();
    }

    function setLanguage($language)
    {
        $this->storage->set('language', $language);
    }

    function getLanguage()
    {
        return $this->storage->get('language');
    }

    // ...
}
?>
{% endhighlight %}

>在应用中使用User类

{% highlight php linenos %}
<?php
$user = new User();
$user->setLanguage('fr');
$user_language = $user->getLanguage();
?>
{% endhighlight %}

>当然这些都已经很不错了，直到你想要更灵活实现。如果你想要改变session cookie的名字，该如何做好？这里有几个可行性例子：

>在User类的构造函数中进行设置

{% highlight php linenos %}
<?php
class User
{
    function __construct()
    {
        $this->storage = new SessionStorage('SESSION_ID');
    }

    // ...
}
?>
{% endhighlight %}

>定义一个外部常量

{% highlight php linenos %}
<?php
define('STORAGE_SESSION_NAME', 'SESSION_ID');

class User
{
    function __construct()
    {
        $this->storage = new SessionStorage(STORAGE_SESSION_NAME);
    }

    // ...
}
?>
{% endhighlight %}

>作为User类构造函数的参数

{% highlight php linenos %}
<?php
class User
{
    function __construct($sessionName)
    {
        $this->storage = new SessionStorage($sessionName);
    }
 
    // ...
}
 
$user = new User('SESSION_ID');
?>
{% endhighlight %}

>所有的这些硬编码方法都显得有些鸡肋（对于灵活性而言）。在User类中设置session name无法真正解决问题，当你当你想改变session name的时候不得不去修改User类。使用常量，使得User类依赖于一个被设置的常量。使用构造函数的参数又显得不对劲，因为User类的构造函数的参数就不是与对象本身相关的了。

>当我们想修改session的存储方式时，如果继续在User类上进行属性或方法的添加，那User类就会显得很臃肿。

>假设我们有一个SessionStorage类，用于实现session机制。

{% highlight php linenos %}
<?php
class SessionStorage
{
    function __construct($cookieName = 'PHP_SESS_ID')
    {
        session_name($cookieName);
        session_start();
    }

    function set($key, $value)
    {
        $_SESSION[$key] = $value;
    }

    function get($key)
    {
        return $_SESSION[$key];
    }

    // ...
}
?>
{% endhighlight %}

>应用依赖注入的思想，可降低程序的耦合度，将SessionStorage类作为User类的依赖，将关于session的事情交由SessionStorage类处理。这样不管是设置session name还是session的存储方式，都不会修改的User类。

>以下是几种依赖注入方式

* Constructor Injection:

{% highlight php linenos %}
<?php
class User
{
    function __construct($storage)
    {
        $this->storage = $storage;
    }

    // ...
}
?>
{% endhighlight %}

* Setter Injection:

{% highlight php linenos %}
<?php
class User
{
    function setSessionStorage($storage)
    {
        $this->storage = $storage;
    }

    // ...
}
?>
{% endhighlight %}

* Property Injection:

{% highlight php linenos %}
<?php
class User
{
    public $sessionStorage;
}
 
$user->sessionStorage = $storage;
?>
{% endhighlight %}