---
layout: post
title: "观察者模式"
description: ""
category: PHP
tags: [PHP设计模式]
---
{% include JB/setup %}

*学习自《PHP 精粹》*

>对于javascript开发者来说,这是一个再熟悉不过的模式了,因为js通过事件驱动来执行,例如页面加载,单击,鼠标移动等.

>观察者模式的核心在于允许你的应用程序注册一个回调,当某个特定的事件发生时便会触发它.

>下面例子通过一个Event类实现观察者模式:

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