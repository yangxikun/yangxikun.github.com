---
layout: post
title: "PHP 协程"
description: ""
category: PHP
tags: []
---
{% include JB/setup %}
#### 理解生成器
- - -
参考官方文档：[Generators](http://php.net/manual/en/language.generators.php)
生成器让我们快速、简单地实现一个迭代器，而不需要创建一个实现了Iterator接口的类后，再实例化出一个对象。

一个生成器长什么样？如下
{% highlight php linenos %}
{% raw %}
<?php
function foo() {
    ……
    yield [$someValue];
    ……
}
{% endraw %}
{% endhighlight %}
与一般函数的区别在于：

* 它不能`return $notNULLValue`（不能有，会报语法错误= =`PHP Fatal error:  Generators cannot return values using "return"`），但可以是`return;`（相当于`return NULL;`其实当一个函数没有明确进行return时，PHP会自动为函数加入`return;`）
* 必须含有yield关键字（当生成器执行的时候，每次执行到yield都会中断，并且将`$someValue`作为返回值，如果有的话，没有则是返回NULL）。yield的具体语法见：[Generator syntax](http://php.net/manual/en/language.generators.syntax.php)
* 它会被转换为Generator类的一个对象

<!--more-->
生成器是如何执行的？

先看看Generator这个类：
{% highlight php linenos %}
{% raw %}
<?php
Generator implements Iterator {//实现了Iterator接口，可被foreach迭代访问
    /* Methods */
    public mixed current ( void )   //获取yielded value
    public mixed key ( void )       //获取yielded key
    public void next ( void )       //从上一次断点之后继续执行
    public void rewind ( void )     //重置迭代
    public mixed send ( mixed $value )          //向生成器中传入$value后，从上一次断点之后继续执行
    public mixed throw ( Exception $exception ) //向生成器中抛入一个异常，从上一次断点之后继续执行
    public bool valid ( void )      //检查迭代是否结束了
    public void __wakeup ( void )   //当序列化Generator对象时抛出异常，即Generator对象不能进行序列化
}
{% endraw %}
{% endhighlight %}
以上方法的具体用法见：[The Generator class](http://php.net/manual/en/class.generator.php)

生成器的执行是在调用`next(), send(), throw()`时就会执行一次。

对于`rewind()`方法，当生成器执行过时，调用该方法将会抛出异常，其文档中的评论有人提到`rewind()`方法存在的原因可能只是为了兼容Iterator接口= =。

另一个需要注意的是调用一个Generator对象的方法时，都会首先判断Generator对象是否初始化了（这里不是指调用构造函数，而是指Generator对象是否执行过第一次了），没有的话就初始化：
{% highlight php linenos %}
{% raw %}
<?php
function gen() {
    $a = 'foo';
    $b = (yield $a);//这里其实可以看做两句语句的执行：return $a; $b = $sendedValue;
    var_dump($b);
    yield 'bar';
}
$gen = gen();
var_dump($gen->current());
var_dump($gen->send('something'));

//output:
//string(3) "foo"
//string(9) "something"
//string(3) "bar"
{% endraw %}
{% endhighlight %}
从Generator的源码就可以发现这一点（Zend/zend_generators.c）：
{% highlight php linenos %}
{% raw %}
<?php
ZEND_METHOD(Generator, current)
{
    ……
    zend_generator_ensure_initialized(generator TSRMLS_CC);//Generator每个方法中都会调用该函数
    ……
}
static void zend_generator_ensure_initialized(zend_generator *generator TSRMLS_DC)
{
    if (generator->execute_data && !generator->value) {//如果当前的value为null，则让Generator执行一次，即执行到第一个yield处中断
        zend_generator_resume(generator TSRMLS_CC);//执行Generator一次
        generator->flags |= ZEND_GENERATOR_AT_FIRST_YIELD;
    }
}
{% endraw %}
{% endhighlight %}
那么生成器执行的生命周期如下图所示：
![](/assets/img/201603200101.png)

一个generator可以看成由多个yield组成，每一次执行都会执行到下一个yield中断并返回产生的值或结束。

其实对于Generator我有两个疑惑：

1. 为何`zend_generator_ensure_initialized(generator TSRMLS_CC)`不在构造函数中执行，而是放到每个方法中；
1. 为何Generator执行过后，调用`rewind()`方法无法重置迭代，是实现不了么= =。

##### 一个关于生成器使用的栗子

如果不用生成器，代码是这样子：
{% highlight php linenos %}
{% raw %}
<?php
foreach (range(1, 10, 1) as $number) {
    echo "$number ";
}
//output: 1 2 3 4 5 6 7 8 9 10
{% endraw %}
{% endhighlight %}
如果用生成器，代码是这样子：
{% highlight php linenos %}
{% raw %}
<?php
function xrange($start, $limit, $step = 1) {
    if ($start < $limit) {
        if ($step <= 0) {
            throw new LogicException('Step must be +ve');
        }

        for ($i = $start; $i <= $limit; $i += $step) {
            yield $i;
        }
    } else {
        if ($step >= 0) {
            throw new LogicException('Step must be -ve');
        }

        for ($i = $start; $i >= $limit; $i += $step) {
            yield $i;
        }
    }
}
foreach (xrange(1, 10, 1) as $number) { //生成器是一个Generator对象，继承了Iterator
    echo "$number ";
}
//output: 1 2 3 4 5 6 7 8 9 10
{% endraw %}
{% endhighlight %}
用了生成器代码量反而翻倍了，其实`range()`的调用会产生一个数组返回，如果产生一个超大数组的话，那么就会占用掉许多内存。而使用生成器的话就不会产生一个超大数组，生成器每次运行到`yield $i`的时候就会中断执行，并且返回`$i`的值。也许有人会说，我这里用一个for循环就搞定了呀！是的，对于上面简单的逻辑是可行的，但如果需要比较复杂的逻辑的话，又需要复用，就可以封装到Generator中了。

#### 理解协程
- - -
维基百科对于协程的定义：[Coroutine](https://en.wikipedia.org/wiki/Coroutine)
协程是作为程序的一个组件，通过允许多个入口点挂起和恢复执行，产生非抢占的多任务处理子例程。

个人理解：就像操作系统中任务是由进程去做的，操作系统负责对它们进行调度。而在一个进程中，如果要处理多个任务，可以交给协程去处理，并由进程进行调度，之所以说是非抢占式是因为协程的终止需要当前执行的协程主动交出CPU的使用权，多个协程是不断地交替串行执行的。

PHP的Generator符合成为协程的要求：允许多个入口点挂起和恢复执行，一个Generator对象可以看作为一个协程。

那么在PHP中要使用协程进行多任务的处理，就还差一个调度器。

鸟哥有一篇文章是讲有关协程的：[在PHP中使用协程实现多任务调度](http://www.laruence.com/2015/05/28/3038.html)（文章的最原始出处应该是[Cooperative multitasking using coroutines (in PHP!)](http://nikic.github.io/2012/12/22/Cooperative-multitasking-using-coroutines-in-PHP.html)） 文中就讲解了如何实现一个调度器，并配合非阻塞IO实现一个简单的回显web服务器。
一开始看鸟哥的这篇文章有很多模糊不清的地方（所以才有了此文= =），下文内容将基于鸟哥的文章，再加上自己的理解阐述PHP的协程，画图思路参考了一篇内部文章：**TSF高性能后台服务解决方案-总体介绍**中的“揭秘TSF：协程”这张图。

#### 实现调度器
- - -
基础版的调度器如下：
![](/assets/img/201603200102.png)

首先实现“任务”——对Generator的封装：
{% highlight php linenos %}
{% raw %}
<?php
class Task {
    protected $taskId;
    protected $coroutine;                   //Generator对象
    protected $sendValue = null;            //发送到Generator中的值
    protected $beforeFirstYield = true;     //用于判断是否第一次执行Generator
 
    public function __construct($taskId, Generator $coroutine) {
        $this->taskId = $taskId;
        $this->coroutine = $coroutine;
    }
 
    public function getTaskId() {
        return $this->taskId;
    }
 
    public function setSendValue($sendValue) {//指定哪些值将被发送到下次的恢复执行中
        $this->sendValue = $sendValue;
    }
 
    public function run() {
        if ($this->beforeFirstYield) {//第一次执行，返回当前的value，上文中已解释过这里的原因
            $this->beforeFirstYield = false;
            return $this->coroutine->current();
        } else {
            $retval = $this->coroutine->send($this->sendValue);
            $this->sendValue = null;
            return $retval;
        }
    }
 
    public function isFinished() {
        return !$this->coroutine->valid();
    }
}
{% endraw %}
{% endhighlight %}
调度器：
{% highlight php linenos %}
{% raw %}
<?php
class Scheduler {
    protected $maxTaskId = 0;
    protected $taskMap = []; // taskId => task
    protected $taskQueue;
 
    public function __construct() {
        $this->taskQueue = new SplQueue();
    }
 
    public function newTask(Generator $coroutine) {
        $tid = ++$this->maxTaskId;
        $task = new Task($tid, $coroutine);
        $this->taskMap[$tid] = $task;
        $this->schedule($task);
        return $tid;
    }
 
    public function schedule(Task $task) {
        $this->taskQueue->enqueue($task);
    }
 
    public function run() {//多个任务是交替串行执行的
        while (!$this->taskQueue->isEmpty()) {//如果队列任务不为空，则不断出队任务，并执行
            $task = $this->taskQueue->dequeue();
            $task->run();
 
            if ($task->isFinished()) {//判断当前任务是否执行结束了
                unset($this->taskMap[$task->getTaskId()]);
            } else {//还没执行结束的任务需要重新入队，以便下次调度
                $this->schedule($task);
            }
        }
    }
}
{% endraw %}
{% endhighlight %}
以两个简单的任务来测试上述调度器：
{% highlight php linenos %}
{% raw %}
<?php
function task1() {//协程1
    for ($i = 1; $i <= 10; ++$i) {
        echo "This is task 1 iteration $i.\n";
        yield;
    }
}
 
function task2() {//协程2
    for ($i = 1; $i <= 5; ++$i) {
        echo "This is task 2 iteration $i.\n";
        yield;
    }
}
 
$scheduler = new Scheduler;
 
$scheduler->newTask(task1());
$scheduler->newTask(task2());
 
$scheduler->run();

//output
/*
This is task 1 iteration 1.
This is task 2 iteration 1.
This is task 1 iteration 2.
This is task 2 iteration 2.
This is task 1 iteration 3.
This is task 2 iteration 3.
This is task 1 iteration 4.
This is task 2 iteration 4.
This is task 1 iteration 5.
This is task 2 iteration 5.
This is task 1 iteration 6.
This is task 1 iteration 7.
This is task 1 iteration 8.
This is task 1 iteration 9.
This is task 1 iteration 10.
*/
{% endraw %}
{% endhighlight %}
输出结果正如期望的：对于前5个迭代来说，两个任务是交替执行的，在第2个任务结束后，只有第一个任务继续执行。

在C语言编程中，进程与内核的通信是通过系统调用实现的，那么协程与调度器之间的通信也是要借助“系统调用”来实现（当然也可以不通过“系统调用”这层抽象来做，只不过加了“系统调用”这层抽象的话代码会优雅很多，因为协程与调度器之间通信时要做的事情可能有很多种，我们可以将不同种类的事情封装到不同的“系统调用”中）。

那么“系统调用”所要做的是什么事情？

1. 产生任务需要的value；
1. 重新调度任务。

先看一个获取任务Id的“系统调用”：
{% highlight php linenos %}
{% raw %}
<?php
function getTaskId() {
    return new SystemCall(function(Task $task, Scheduler $scheduler) {
        $task->setSendValue($task->getTaskId());//产生任务需要的value，在task->run()时会通过task->coroutine->send(task->sendValue)发送到Generator中
        $scheduler->schedule($task);//重新调度任务
    });
}
{% endraw %}
{% endhighlight %}
`getTaskId()`返回了一个`SystemCall`类的对象，构造方法的参数中传入了一个回调函数。

封装“系统调用”的类如下：
{% highlight php linenos %}
{% raw %}
<?php
class SystemCall {
    protected $callback;
 
    public function __construct(callable $callback) {
        $this->callback = $callback;
    }
 
    public function __invoke(Task $task, Scheduler $scheduler) {//当SystemCall对象被当做函数调用时调用该方法
        $callback = $this->callback;
        return $callback($task, $scheduler);
    }
}
{% endraw %}
{% endhighlight %}
现在，需要修改下调度器的`run()`方法以支持“系统调用”：
{% highlight php linenos %}
{% raw %}
<?php
public function run() {
    while (!$this->taskQueue->isEmpty()) {
        $task = $this->taskQueue->dequeue();
        $retval = $task->run();
 
        if ($retval instanceof SystemCall) {//如果任务发出了一个系统调用，那么执行该系统调用
            $retval($task, $this);
            continue;//因为在系统调用中已经重新调度task，并且此时任务肯定还未执行结束，所以直接continue
        }
 
        if ($task->isFinished()) {
            unset($this->taskMap[$task->getTaskId()]);
        } else {
            $this->schedule($task);
        }
    }
}
{% endraw %}
{% endhighlight %}
测试下“系统调用”：
{% highlight php linenos %}
{% raw %}
<?php
function task($max) {
    $tid = (yield getTaskId()); //发出一个系统调用，这里可以看成：return getTaskId();$tid = task->sendValue;
    for ($i = 1; $i <= $max; ++$i) {
        echo "This is task $tid iteration $i.\n";
        yield;
    }
}
 
$scheduler = new Scheduler;
 
$scheduler->newTask(task(10));
$scheduler->newTask(task(5));
 
$scheduler->run();

//output: 与上个例子相同
{% endraw %}
{% endhighlight %}

再加入2个“系统调用”：
{% highlight php linenos %}
{% raw %}
<?php
function newTask(Generator $coroutine) {//产生新任务的“系统调用”
    return new SystemCall(
        function(Task $task, Scheduler $scheduler) use ($coroutine) {
            $task->setSendValue($scheduler->newTask($coroutine));//返回taskId给调用者
            $scheduler->schedule($task);//重新调度发出当前“系统调用”的任务
        }
    );
}
function killTask($tid) {//杀死某个任务
    return new SystemCall(
        function(Task $task, Scheduler $scheduler) use ($tid) {
            $task->setSendValue($scheduler->killTask($tid));//成功杀死某个任务返回true，否则返回false
            $scheduler->schedule($task);//重新调度发出当前“系统调用”的任务
        }
    );
}
{% endraw %}
{% endhighlight %}
在调度器中需要新增一个`killTask()`方法：
{% highlight php linenos %}
{% raw %}
<?php
public function killTask($tid) {
    if (!isset($this->taskMap[$tid])) {//判断tid是否存在
        return false;
    }
 
    unset($this->taskMap[$tid]);
 
    // This is a bit ugly and could be optimized so it does not have to walk the queue,
    // but assuming that killing tasks is rather rare I won't bother with it now
    foreach ($this->taskQueue as $i => $task) {//遍历任务队列找到tid后从队列中去除它
        if ($task->getTaskId() === $tid) {
            unset($this->taskQueue[$i]);
            break;
        }
    }
 
    return true;
}
{% endraw %}
{% endhighlight %}
测试下新增的“系统调用”：
{% highlight php linenos %}
{% raw %}
<?php
function childTask() {//子任务
    $tid = (yield getTaskId());
    while (true) {
        echo "Child task $tid still alive!\n";
        yield;
    }
}
 
function task() {//父任务
    $tid = (yield getTaskId());
    $childTid = (yield newTask(childTask()));//产生一个子任务，并获取返回的子任务id
 
    for ($i = 1; $i <= 6; ++$i) {
        echo "Parent task $tid iteration $i.\n";
        yield;
 
        if ($i == 3) yield killTask($childTid);//杀死子任务
    }
}
 
$scheduler = new Scheduler;
$scheduler->newTask(task());
$scheduler->run();

//output:
/*
Parent task 1 iteration 1.
Child task 2 still alive!
Parent task 1 iteration 2.
Child task 2 still alive!
Parent task 1 iteration 3.
Child task 2 still alive!
Parent task 1 iteration 4.
Parent task 1 iteration 5.
Parent task 1 iteration 6.
*/
{% endraw %}
{% endhighlight %}
可以看到在第3次迭代之后子任务被杀死了，注意这里的父子关系只是逻辑意义上的。

该调度器的示例代码目标是完成一个web服务器：一个任务负责在套接字上监听是否有新的连接，当有新连接要建立的时候，它创建一个新任务来处理新连接，对于套接字的IO都是非阻塞的。

首先，新增两个新的“系统调用”：
{% highlight php linenos %}
{% raw %}
<?php
function waitForRead($socket) {//等待socket可以读
    return new SystemCall(
        function(Task $task, Scheduler $scheduler) use ($socket) {
            $scheduler->waitForRead($socket, $task);//将task加入等待socket可以读的队列中
        }
    );
}
 
function waitForWrite($socket) {//等待socket可以写
    return new SystemCall(
        function(Task $task, Scheduler $scheduler) use ($socket) {
            $scheduler->waitForWrite($socket, $task);//将task加入等待socket可以写的队列中
        }
    );
}
{% endraw %}
{% endhighlight %}
调度器需要新增两个入事件队列的方法：
{% highlight php linenos %}
{% raw %}
<?php
// resourceID => [socket, [tasks]]
protected $waitingForRead = []; //等待socket可以读的队列
protected $waitingForWrite = [];//等待socket可以写的队列
 
public function waitForRead($socket, Task $task) {
    if (isset($this->waitingForRead[(int) $socket])) {
        $this->waitingFo
        
        rRead[(int) $socket][1][] = $task;//入队
    } else {
        $this->waitingForRead[(int) $socket] = [$socket, [$task]];//初始化队列
    }
}
 
public function waitForWrite($socket, Task $task) {
    if (isset($this->waitingForWrite[(int) $socket])) {
        $this->waitingForWrite[(int) $socket][1][] = $task;
    } else {
        $this->waitingForWrite[(int) $socket] = [$socket, [$task]];
    }
}
{% endraw %}
{% endhighlight %}
在Linux C中，epoll负责监听多个文件描述符上发生的事件，在PHP中可以用`stream_select()`函数模拟epoll，下面将在调度器中增加处理socket读写事件的方法：
{% highlight php linenos %}
{% raw %}
<?php
protected function ioPoll($timeout) {
    $rSocks = [];
    foreach ($this->waitingForRead as list($socket)) {
        $rSocks[] = $socket;
    }
 
    $wSocks = [];
    foreach ($this->waitingForWrite as list($socket)) {
        $wSocks[] = $socket;
    }
 
    $eSocks = []; // dummy
 
    if (!stream_select($rSocks, $wSocks, $eSocks, $timeout)) {
        return;
    }
 
    foreach ($rSocks as $socket) {//读事件发生
        list(, $tasks) = $this->waitingForRead[(int) $socket];
        unset($this->waitingForRead[(int) $socket]);//从队列中去除
 
        foreach ($tasks as $task) {//逐个调度等待该socket上发生的读事件的任务
            $this->schedule($task);
        }
    }
 
    foreach ($wSocks as $socket) {//写事件发生
        list(, $tasks) = $this->waitingForWrite[(int) $socket];
        unset($this->waitingForWrite[(int) $socket]);//从队列中去除
 
        foreach ($tasks as $task) {//逐个调度等待该socket上发生的写事件的任务
            $this->schedule($task);
        }
    }
}
{% endraw %}
{% endhighlight %}
接下来需要在合适的地方调用`ioPoll()`方法，可以在调度器调度某个task之后，调用该方法：
{% highlight php linenos %}
{% raw %}
<?php
public function run() {
    while (!$this->taskQueue->isEmpty()) {
        $task = $this->taskQueue->dequeue();
        $retval = $task->run();
        if ($retval instanceof SystemCall) {
            $retval($task, $this);
            if ($this->taskQueue->isEmpty()) {//如果任务队列为空的话，则阻塞stream_select的调用，直到有新的连接到来
                $this->ioPoll(null);
            } elseif (!empty($this->waitingForRead) || !empty($this->waitingForWrite)) {
                    //若socket等待队列不为空
                    $this->ioPoll(0);//0秒超时，既不阻塞stream_select的调用
            }
            continue;
        }
 
        if ($this->taskQueue->isEmpty()) {
            $this->ioPoll(null);
        } elseif (!empty($this->waitingForRead) || !empty($this->waitingForWrite)) {
                $this->ioPoll(0);
        }
        if ($task->isFinished()) {
            unset($this->taskMap[$task->getTaskId()]);
        } else {
            $this->schedule($task);
        }
    }
}
{% endraw %}
{% endhighlight %}
以上方法并不太好，因为每次调度任务的时候都要做一次逻辑判断以及`ioPoll()`方法调用会比较频繁，比较好的的做法是把`ioPoll()`方法封装到一个task任务中去执行（就好像一个“常驻任务”，每次调用到它时就去调用`ioPoll()`方法查看是否有socket事件发生）：
{% highlight php linenos %}
{% raw %}
<?php
protected function ioPollTask() {
    while (true) {
        if ($this->taskQueue->isEmpty()) {
            $this->ioPoll(null);//阻塞stream_select的调用，直到有新的连接到来
        } else {
            $this->ioPoll(0);//0秒超时，既不阻塞stream_select的调用
        }
        yield;//将yield放到无限循环中，使得该task能“常驻”在调度器队列中
    }
}

public function run() {
    $this->newTask($this->ioPollTask());//产生“常驻任务”，负责检查socket事件
    while (!$this->taskQueue->isEmpty()) {
        $task = $this->taskQueue->dequeue();
        $retval = $task->run();
        if ($retval instanceof SystemCall) {
            $retval($task, $this);
            continue;
        }
 
        if ($task->isFinished()) {
            unset($this->taskMap[$task->getTaskId()]);
        } else {
            $this->schedule($task);
        }
    }
}
{% endraw %}
{% endhighlight %}
这个时候就该测试下前面完成的成果了：
{% highlight php linenos %}
{% raw %}
<?php
 
function server($port) {
    echo "Starting server at port $port...\n";
 
    $socket = @stream_socket_server("tcp://localhost:$port", $errNo, $errStr);
    if (!$socket) throw new Exception($errStr, $errNo);
 
    stream_set_blocking($socket, 0);//设置socket非阻塞
 
    while (true) {
        yield waitForRead($socket);
        $clientSocket = stream_socket_accept($socket, 0);//accept的socket也是非阻塞的
        yield newTask(handleClient($clientSocket));//产生新的任务处理客户端请求
    }
}
 
function handleClient($socket) {//负责处理客户端请求的Generator
    yield waitForRead($socket);
    $data = fread($socket, 8192);
 
    $msg = "Received following request:\n\n$data";
    $msgLength = strlen($msg);
 
    $response = <<<RES
HTTP/1.1 200 OK\r
Content-Type: text/plain\r
Content-Length: $msgLength\r
Connection: close\r
\r
$msg
RES;
 
    yield waitForWrite($socket);
    fwrite($socket, $response);
 
    fclose($socket);
}
 
$scheduler = new Scheduler;
$scheduler->newTask(server(8000));
$scheduler->run();
{% endraw %}
{% endhighlight %}
上述测试代码接收`localhost:8000`上的连接，然后返回发送来的内容作为HTTP响应。
可以使用`curl http://localhost:8000/`或`ab -n 20 -c 20 localhost:8000/`进行访问测试。

现在新版调度器如下：
![](/assets/img/201603200103.png)

## 协程栈

假如在一个协程（Generator对象）中有另外一个协程需要执行，按照之前完成的调度器是无法正确工作的：
{% highlight php linenos %}
{% raw %}
<?php

function getSomething() //child coroutine
{
    ……
    yield
    ……
}

function task()
{
    $v = (yield getSomething());    //希望在这里调用子协程，并获得返回值
    ……
    yield
}

$scheduler = new Scheduler;
$scheduler->newTask(task());
$scheduler->run();
{% endraw %}
{% endhighlight %}
注意上面代码中的`$v = (yield getSomething());`，这里跟之前的那些task的不同之处在于其yield抛出的值是协程，而不是“系统调用”。
那么这个时候就需要一个类似如下图的协程堆栈调用：
![](/assets/img/201603200104.png)

首先，实现对子协程返回值的封装：
{% highlight php linenos %}
{% raw %}
<?php
 
class CoroutineReturnValue {
    protected $value;
 
    public function __construct($value) {
        $this->value = $value;
    }
 
    public function getValue() {
        return $this->value;
    }
}
 
function retval($value) {
    return new CoroutineReturnValue($value);
}
{% endraw %}
{% endhighlight %}

实现协程栈：
{% highlight php linenos %}
{% raw %}
<?php

function stackedCoroutine(Generator $gen) {//本身也是个协程
    $stack = new SplStack;
 
    for (;;) {
        $value = $gen->current();
 
        if ($value instanceof Generator) {//如果是子协程调用
            $stack->push($gen);//父协程压栈
            $gen = $value;//$gen指向当前协程
            continue;
        }
 
        $isReturnValue = $value instanceof CoroutineReturnValue;
        if (!$gen->valid() || $isReturnValue) {//如果当前协程执行完了或者当前协程返回的值是CoroutineReturnValue对象（即该值是要返回给父协程的），则结束当前协程的执行
            if ($stack->isEmpty()) {//如果栈空了，那么整个协程栈执行完毕
                return;
            }
 
            $gen = $stack->pop();//如果协程栈不为空，那么弹出下一个需要执行的协程
            $gen->send($isReturnValue ? $value->getValue() : NULL);//如果子协程有返回值给到父协程，则发送过去
            continue;
        }
 
        $gen->send(yield $gen->key() => $value);
    }
}
{% endraw %}
{% endhighlight %}

Task类中的构造函数需要做一点点修改：
{% highlight php linenos %}
{% raw %}
<?php
public function __construct($taskId, Generator $coroutine) {
    $this->taskId = $taskId;
    $this->coroutine = StackedCoroutine($coroutine);    //coroutine属性赋值为协程栈
}
{% endraw %}
{% endhighlight %}

现在可以改进下上面的web服务器例子，把socket相关操作都封装到一个类中：
{% highlight php linenos %}
{% raw %}
<?php
 
class CoSocket {
    protected $socket;
 
    public function __construct($socket) {
        $this->socket = $socket;//socket资源
    }
 
    public function accept() {
        yield waitForRead($this->socket);
        yield retval(new CoSocket(stream_socket_accept($this->socket, 0)));
    }
 
    public function read($size) {
        yield waitForRead($this->socket);
        yield retval(fread($this->socket, $size));
    }
 
    public function write($string) {
        yield waitForWrite($this->socket);
        fwrite($this->socket, $string);
    }
 
    public function close() {
        @fclose($this->socket);
    }
}
{% endraw %}
{% endhighlight %}

现在server端可以这样写了：
{% highlight php linenos %}
{% raw %}
<?php
 
function server($port) {
    echo "Starting server at port $port...\n";
 
    $socket = @stream_socket_server("tcp://localhost:$port", $errNo, $errStr);
    if (!$socket) throw new Exception($errStr, $errNo);
 
    stream_set_blocking($socket, 0);
 
    $socket = new CoSocket($socket);
    while (true) {
        //yield $socket->accept() 是子协程调用
        yield newTask(
            handleClient(yield $socket->accept())//这里handleClient()函数接收到的参数是一个CoSocket对象
        );
    }
}
 
function handleClient($socket) {
    $data = (yield $socket->read(8192));//子协程调用
 
    $msg = "Received following request:\n\n$data";
    $msgLength = strlen($msg);
 
    $response = <<<RES
HTTP/1.1 200 OK\r
Content-Type: text/plain\r
Content-Length: $msgLength\r
Connection: close\r
\r
$msg
RES;
 
    yield $socket->write($response);
    yield $socket->close();
}

{% endraw %}
{% endhighlight %}

#### 错误处理
- - -
现在整个web服务器还缺少对出错的处理，无法在需要的时候抛出异常，为了解决这个问题，可以使用Generator的`throw()`方法。

`throw()`方法接受一个`Exception`, 并将其抛出到协程的当前悬挂点, 看看下面代码：
{% highlight php linenos %}
{% raw %}
<?php
function gen() {
    echo "Foo\n";
    try {
        yield;
    } catch (Exception $e) {
        echo "Exception: {$e->getMessage()}\n";
    }
    echo "Bar\n";
}
 
$gen = gen();
$gen->rewind();                     // echos "Foo"
$gen->throw(new Exception('Test')); // echos "Exception: Test"
                                    // and "Bar"
{% endraw %}
{% endhighlight %}

这样的话，当在进行“系统调用”和子协程调用时就可以抛出异常了。
首先需要修改`Scheduler::run()`方法：
{% highlight php linenos %}
{% raw %}
<?php
if ($retval instanceof SystemCall) {
    try {
        $retval($task, $this);
    } catch (Exception $e) {
        $task->setException($e);
        $this->schedule($task);
    }
    continue;
}
{% endraw %}
{% endhighlight %}

再修改Task类以支持异常传递到协程中：
{% highlight php linenos %}
{% raw %}
<?php
class Task {
    // ...
    protected $exception = null;
 
    public function setException($exception) {
        $this->exception = $exception;
    }
 
    public function run() {
        if ($this->beforeFirstYield) {
            $this->beforeFirstYield = false;
            return $this->coroutine->current();
        } elseif ($this->exception) {//如果当前exception属性非null，那么向协程栈中抛入该异常
            $retval = $this->coroutine->throw($this->exception);
            $this->exception = null;
            return $retval;
        } else {
            $retval = $this->coroutine->send($this->sendValue);
            $this->sendValue = null;
            return $retval;
        }
    }
 
    // ...
}
{% endraw %}
{% endhighlight %}

最后，还需要修改协程栈，以正确处理异常：
{% highlight php linenos %}
{% raw %}
<?php
function stackedCoroutine(Generator $gen) {
    $stack = new SplStack;
    $exception = null;
 
    for (;;) {
        try {
            if ($exception) {
                $gen->throw($exception);
                $exception = null;
                continue;
            }
 
            $value = $gen->current();
 
            if ($value instanceof Generator) {
                $stack->push($gen);
                $gen = $value;
                continue;
            }
 
            $isReturnValue = $value instanceof CoroutineReturnValue;
            if (!$gen->valid() || $isReturnValue) {
                if ($stack->isEmpty()) {
                    return;
                }
 
                $gen = $stack->pop();
                $gen->send($isReturnValue ? $value->getValue() : NULL);
                continue;
            }
 
            try {
                $sendValue = (yield $gen->key() => $value);//处理“系统调用”抛出的Exception
            } catch (Exception $e) {
                $gen->throw($e);
                continue;
            }
 
            $gen->send($sendValue);
        } catch (Exception $e) {//捕获try代码块中的$gen执行时抛出的异常
            if ($stack->isEmpty()) {
                throw $e;
            }
 
            $gen = $stack->pop();
            $exception = $e;
        }
    }
}
{% endraw %}
{% endhighlight %}

现在，可以在“系统调用”中使用异常抛出了！例如，调用killTask时，当传递一个不存在的tid时，抛出一个异常：
{% highlight php linenos %}
{% raw %}
<?php
function killTask($tid) {
    return new SystemCall(
        function(Task $task, Scheduler $scheduler) use ($tid) {
            if ($scheduler->killTask($tid)) {
                $scheduler->schedule($task);
            } else {
                throw new InvalidArgumentException('Invalid task ID!');
            }
        }
    );
}
{% endraw %}
{% endhighlight %}

测试代码（记得注释掉，此前在`Scheduler::run()`方法中设置的“常驻任务”：`$this->newTask($this->ioPollTask());`）：
{% highlight php linenos %}
{% raw %}
<?php
function task() {
    try {
        yield killTask(500);
    } catch (Exception $e) {
        echo 'Tried to kill task 500 but failed: ', $e->getMessage(), "\n";
    }
}
{% endraw %}
{% endhighlight %}

## 结语
当我们重新审视协程(Task)中的代码时，会发现编写逻辑跟同步编码十分相似，然而，很酷的事情是通过`yield`发起的操作实际上都是异步执行的！

举个例子说明下协程的用处，例如现在我们的业务代码里需要调用（通过tcp、udp、http……）其他业务系统的服务，该调用会有一定的耗时。
如果使用PHP语言常规的LNMP模式，当客户端访问我们的业务代码时，能同时处理的请求数 = php-fpm开启的进程数。
如果使用PHP的协程（当然前提是要有一个功能完备且强大的调度器），那么同时处理的请求数 = php进程数 * 每个进程能开启的协程数。

支持协程的语言不止是PHP，还有很多其他语言都有相应的机制实现协程，参阅：[Coroutine](https://en.wikipedia.org/wiki/Coroutine)