---
layout: post
title: "UML类图关系：继承、实现、依赖、关联、聚合、组合"
description: ""
category: 软件工程
tags: [UML]
---
{% include JB/setup %}

*学习自：[UML中几种类间关系：继承、实现、依赖、关联、聚合、组合的联系与区别](http://blog.csdn.net/sfdev/article/details/3906243) 和 《UML和模式应用》*

##### 继承

一个类继承另外一个类，或者一个接口继承了其他接口。

![](/assets/img/201403270101.png)

{% highlight php linenos %}
@startuml
ClassA <|-- ClassB
@enduml
{% endhighlight %}

<!--more-->
##### 实现

一个类实现了一个或多个接口。

![](/assets/img/201403270102.png)

{% highlight php linenos %}
@startuml
ClassA <|.. ClassB
@enduml
{% endhighlight %}

##### 依赖

一个类A使用到了另一个类B，而这种使用关系是具有偶然性的、、临时性的、非常弱的，且类B的变化会影响到类A。比如某人要过河，需要借用一条船，此时人与船之间的关系就是依赖。表现在代码层面：

* 类B作为类A某个method的参数变量；
* 类B作为类A某个method的局部变量；
* 类A调用了类B的某个静态method。

![](/assets/img/201403270103.png)

{% highlight php linenos %}
@startuml
ClassA ..> ClassB
@enduml
{% endhighlight %}

##### 关联

类与类之间的连结，关联关系使一个类知道另外一个类的属性和方法；通常含有“知道”，“了解”的含义类与类之间的连结，关联关系使一个类知道另外一个类的属性和方法；通常含有“知道”，“了解”的含义。关联可以是单向、双向的。比如渔夫要出海打鱼，得先了解下天气情况。表现在代码层面，为被关联类B以类属性的形式出现在关联类A中，也可能是关联类A引用了一个类型为被关联类B的全局变量。

![](/assets/img/201403270104.png)

{% highlight php linenos %}
@startuml
ClassA --> "1" ClassB
@enduml
{% endhighlight %}

##### 聚合

是关联关系的一种，是一种强关联关系；聚合关系是整体和个体/部分之间的关系；关联关系的两个类处于同一个层次上，而聚合关系的两个类处于不同的层次上，一个是整体，一个是个体/部分；在聚合关系中，代表个体/部分的对象有可能会被多个代表整体的对象所共享。比如：学生组织和学生之间的关系。

![](/assets/img/201403270105.png)

{% highlight php linenos %}
@startuml
StudentOrganization o--> "1..*" Student
@enduml
{% endhighlight %}

##### 组合

它也是关联关系的一种，但它是比聚合关系更强的关系.组合关系要求聚合关系中代表整体的对象要负责代表个体/部分的对象的整个生命周期；组合关系不能共享；在组合关系中，如果代表整体的对象被销毁或破坏，那么代表个体/部分的对象也一定会被销毁或破坏，而聚在合关系中，代表个体/部分的对象则有可能被多个代表整体的对象所共享，而不一定会随着某个代表整体的对象被销毁或破坏而被销毁或破坏。比如人与自己的手的关系。

![](/assets/img/201403270106.png)

{% highlight php linenos %}
@startuml
Person *--> "1..2" hand
@enduml
{% endhighlight %}

几种关系所表现的强弱程度依次为：组合>聚合>关联>依赖。