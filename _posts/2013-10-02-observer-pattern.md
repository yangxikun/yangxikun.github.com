---
layout: post
title: "观察者模式"
description: ""
category: PHP
tags: [PHP设计模式]
---
{% include JB/setup %}

*学习自《PHP 精粹》*

对于javascript开发者来说,这是一个再熟悉不过的模式了,因为js通过事件驱动来执行,例如页面加载,单击,鼠标移动等.

观察者模式的核心在于允许你的应用程序注册一个回调,当某个特定的事件发生时便会触发它.

下面例子通过一个Event类实现观察者模式:

{% highlight php linenos %}
<?php
/**
 * Event 类
 * 
 * 使用Event类,你可以为某个事件注册回调函数
 */
class Event {

    /**
     * @static
     * @var array 键值对数组,键作为事件,值是存储事件的回调数组
     */
    static protected $callbacks = array();

    /**
     * 注册一个回调
     * 
     * @param string $eventName 事件的名字
     * @param mixed $callback 回调
     * @return NULL
     * @throws  Exception
     */
    static public function registerCallback($eventName, $callback) {
        if (!is_callable($callback)) {
            throw new Exception('Invalid callback');
        }
        $eventName = strtolower($eventName);
        self::$callbacks[$eventName][] = $callback;
    }

    /**
     * 触发一个事件
     * 
     * @param string $eventName 事件名称
     * @param mixed $data 传递给回调函数的数据
     * @return NULL
     * @throws  Exception
     */
    static public function trigger($eventName, $data) {
        $eventName = strtolower($eventName);
        if (isset(self::$callbacks[$eventName])) {
            foreach (self::$callbacks[$eventName] as $key => $callback) {
                $callback($data);
            }
        } else {
            throw new Exception('Invalid event');
        }
    }
}

/**
 * log回调
 */
class LogCallback {
    public function __invoke($data) {
        echo 'Log data<br />';
        var_dump($data);
    }
}

//注册回调
Event::registerCallback('save', new LogCallback());
//注册一个闭包
Event::registerCallback('save', function ($data) {
                                    echo 'Clear Cache<br />';
                                    var_dump($data);
                                });
class DataRecord {
    public function save() {//触发事件
        Event::trigger('save', array('Hello', 'world'));
    }
}

$data = new DataRecord();
$data->save();
?>
{% endhighlight %}

GitHub上一个PHPDesignPattern的项目使用了SPL接口实现的观察者模式：

目的：为了实现发布/订阅行为的对象，每当一个“目标”对象改变它的状态，附加的“观察员”将被通报。它是用来缩短耦合对象的量，并使用松散的耦合来代替。

观察者：
{% highlight php linenos %}
<?php

namespace DesignPatterns\Observer;

/**
 * class UserObserver
 */
class UserObserver implements \SplObserver
{
    /**
     * This is the only method to implement as an observer.
     * It is called by the Subject (usually by SplSubject::notify() )
     * 
     * @param \SplSubject $subject
     */
    public function update(\SplSubject $subject)
    {
        echo get_class($subject) . ' has been updated';
    }
}
?>
{% endhighlight %}

“目标”对象：
{% highlight php linenos %}
<?php

namespace DesignPatterns\Observer;

/**
 * Observer pattern : The observed object (the subject)
 * 
 * The subject maintains a list of Observers and sends notifications.
 *
 */
class User implements \SplSubject
{
    /**
     * user data
     *
     * @var array
     */
    protected $data = array();

    /**
     * observers
     *
     * @var array
     */
    protected $observers = array();

    /**
     * attach a new observer
     *
     * @param \SplObserver $observer
     *
     * @return void
     */
    public function attach(\SplObserver $observer)
    {
        $this->observers[] = $observer;
    }

    /**
     * detach an observer
     *
     * @param \SplObserver $observer
     *
     * @return void
     */
    public function detach(\SplObserver $observer)
    {
        $index = array_search($observer, $this->observers);

        if (false !== $index) {
            unset($this->observers[$index]);
        }
    }

    /**
     * notify observers
     *
     * @return void
     */
    public function notify()
    {
        /** @var SplObserver $observer */
        foreach ($this->observers as $observer) {
            $observer->update($this);
        }
    }

    /**
     * Ideally one would better write setter/getter for all valid attributes and only call notify()
     * on attributes that matter when changed
     *
     * @param string $name
     * @param mixed  $value
     *
     * @return void
     */
    public function __set($name, $value)
    {
        $this->data[$name] = $value;

        // notify the observers, that user has been updated
        $this->notify();
    }
}
?>
{% endhighlight %}