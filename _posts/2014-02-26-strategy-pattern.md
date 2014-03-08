---
layout: post
title: "策略模式"
description: ""
category: PHP
tags: [PHP设计模式]
---
{% include JB/setup %}

*学习自[DesignPatternsPHP](https://github.com/domnikl/DesignPatternsPHP)*

#### 内容

根据不同策略，执行不同操作。

#### 特点

将要执行的操作封装起来，能够方便的替换、添加、删除操作。
<!--more-->

#### UML图

![strategyUml](/assets/img/201403080101.svg)

{% highlight php linenos %}
@startuml
interface ComparatorInterface {
    +compare(a:mixed, b:mixed)
}
class DateComparator {
    +compare(a:mixed, b:mixed)
}
class IdComparator {
    +compare(a:mixed, b:mixed)
}
class ObjectCollection {
    +__construct(elements:array)
    +sort() : array
    +setComparator(comparator: ComparatorInterface)
}

ComparatorInterface <|.. DateComparator
ComparatorInterface <|.. IdComparator
ObjectCollection ..> ComparatorInterface
@enduml
{% endhighlight %}

#### 示例代码（源自[DesignPatternsPHP](https://github.com/domnikl/DesignPatternsPHP)）

{% highlight php linenos %}
<?php

namespace DesignPatterns\Strategy;

/**
 * Class ObjectCollection
 */
class ObjectCollection
{
    /**
     * @var array
     */
    private $elements;

    /**
     * @var ComparatorInterface
     */
    private $comparator;

    /**
     * @param array $elements
     */
    public function __construct(array $elements = array())
    {
        $this->elements = $elements;
    }

    /**
     * @return array
     */
    public function sort()
    {
        if (!$this->comparator){ 
            throw new \LogicException("Comparator is not set");    
        }
        
        $callback = array($this->comparator, 'compare');
        uasort($this->elements, $callback);

        return $this->elements;
    }

    /**
     * @param ComparatorInterface $comparator
     *
     * @return void
     */
    public function setComparator(ComparatorInterface $comparator)
    {
        $this->comparator = $comparator;
    }
}
?>
{% endhighlight %}

{% highlight php linenos %}
<?php
namespace DesignPatterns\Strategy;

/**
 * Class ComparatorInterface
 */
interface ComparatorInterface
{
    /**
     * @param mixed $a
     * @param mixed $b
     *
     * @return bool
     */
    public function compare($a, $b);
}
?>
{% endhighlight %}

{% highlight php linenos %}
<?php
namespace DesignPatterns\Strategy;

/**
 * Class DateComparator
 */
class DateComparator implements ComparatorInterface
{
    /**
     * {@inheritdoc}
     */
    public function compare($a, $b)
    {
        $aDate = strtotime($a['date']);
        $bDate = strtotime($b['date']);

        if ($aDate == $bDate) {
            return 0;
        } else {
            return $aDate < $bDate ? -1 : 1;
        }
    }
}
?>
{% endhighlight %}

{% highlight php linenos %}
<?php
namespace DesignPatterns\Strategy;

/**
 * Class IdComparator
 */
class IdComparator implements ComparatorInterface
{
    /**
     * {@inheritdoc}
     */
    public function compare($a, $b)
    {
        if ($a['id'] == $b['id']) {
            return 0;
        } else {
            return $a['id'] < $b['id'] ? -1 : 1;
        }
    }
}
?>
{% endhighlight %}

{% highlight php linenos %}
<?php
namespace DesignPatterns\Strategy;

$elements = array(
    array(
        'id' => 2,
        'date' => '2011-01-01',
    ),
    array(
        'id' => 1,
        'date' => '2011-02-01'
    )
);

$collection = new ObjectCollection($elements);
$collection->setComparator(new IdComparator());
$collection->sort();

$collection->setComparator(new DateComparator());
$collection->sort();
?>
{% endhighlight %}