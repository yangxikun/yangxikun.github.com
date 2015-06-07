---
layout: post
title: "Laravel service providers/service container/facades"
description: ""
category: Laravel
tags: []
---
{% include JB/setup %}

#### 控制反转（IOC）
- - -
为了实现应用中组件的配置与使用分离。

例如，我们有一个应用程序叫Text Editor，它有一个组件叫Spell Checking，那么我们的代码可能会像如下：
{% highlight php linenos %}
<?php
class TextEditor
{
    private SpellChecker checker;
    public __construct() {
        checker = new SpellChecker();
    }
}
?>
{% endhighlight %}
这样的话，TextEditor与SpellChecker就耦合在一起了，TextEditor->checker的实例化完全被TextEditor控制。

<!--more-->

在IOC中，代码会像如下：
{% highlight php linenos %}
<?php
class TextEditor
{
    private ISpellChecker checker;
    public __construct(ISpellChecker checker) {
        checker = checker;
    }
}
?>
{% endhighlight %}
TextEditor->checker的实例化交给了调用者控制了，只要实现了ISpellChecker接口的类就能复制给TextEditor->checker。这就是控制反转，同时也实现了组件的配置和使用分离。

#### 依赖注入（DI）
- - -
是实现IOC的一种设计模式，常见的注入方式：

* A constructor injection
* A parameter injection
* A setter injection
* An interface injection

#### Service container
- - -
Laravel文档中“The Laravel service container is a powerful tool for managing class dependencies. ”也指明了 Service container就是用于依赖管理的，通过依赖注入（DI）的方式实现IOC。

一个讲得不错的视频：[The Service Container](https://laracasts.com/series/laravel-5-fundamentals/episodes/26)

#### Service providers
- - -
Service providers是一个主要配置应用的地方，包括service container bindings, event listeners, filters, even routes。

#### Facades
- - -
应用了设计模式中的外观模式（Facade pattern），目的是为子系统中的一组接口提供一个统一的高层接口，使得子系统更容易使用。从文档中也可以看出来，我们能够方便的FacadeClass::method()去调用。
