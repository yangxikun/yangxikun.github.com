---
layout: post
title: "PHP新的垃圾回收机制"
description: ""
category: PHP
tags: [PHP底层]
---
{% include JB/setup %}

*参考自[PHP新的垃圾回收机制:Zend GC详解](http://blog.csdn.net/phpkernel/article/details/5734743)和[引用计数基本知识](http://php.net/manual/zh/features.gc.refcounting-basics.php)*

###什么算垃圾?

>当一个变量的refcount_gc为0时,将会被作为垃圾进行回收处理.

###新的垃圾回收机制所要解决的问题: 顽固垃圾

>什么是顽固垃圾?

>>示例:当我们添加一个数组本身作为这个数组的元素时,代码如下:

{% highlight php %}
<?php
$a = array( 'one' );
$a[] =& $a;
xdebug_debug_zval( 'a' );
?>
{% endhighlight %}

>>代码输出:

{% highlight php %}
<?php
a: (refcount=2, is_ref=1)=array (
   0 => (refcount=1, is_ref=0)='one',
  1 => (refcount=2, is_ref=1)=...
)
?>
{% endhighlight %}

>>引用手册中的图解:
![PHPGC](/assets/img/201308240101.png)

>能看到数组变量 (a) 同时也是这个数组的第二个元素(1) 指向的变量容器中“refcount”为 2。上面的输出结果中的"..."说明发生了递归操作, 显然在这种情况下意味着"..."指向原始数组。 

>对一个变量调用unset，将删除这个符号，且它指向的变量容器中的引用次数也减1。所以，如果我们在执行完上面的代码后，对变量$a调用unset, 那么变量 $a 和数组元素 "1" 所指向的变量容器的引用次数减1, 从"2"变成"1". 下例可以说明: 

{% highlight php %}
<?php
(refcount=1, is_ref=1)=array (
   0 => (refcount=1, is_ref=0)='one',
  1 => (refcount=1, is_ref=1)=...
)
?>
{% endhighlight %}

>>引用手册中的图解:
![PHPCG](/assets/img/201308240102.png)

>所以新的垃圾回收机制要处理的就是类似以上示例的顽固垃圾.

###新的GC算法

>在较新的PHP手册中有简单的介绍新的GC使用的垃圾清理算法，这个算法名为**Concurrent Cycle Collection in Reference Counted Systems** 这里不详细介绍此算法，根据手册中的内容来先简单的介绍一下思路：

>首先我们有几个基本的准则：

>>1. 如果一个zval的refcount_gc增加，那么此zval还在使用，不属于垃圾
>>2. 如果一个zval的refcount_gc减少到0， 那么zval可以被释放掉，不属于垃圾
>>3. 如果一个zval的refcount_gc减少之后大于0，那么此zval还不能被释放，此zval可能成为一个垃圾

>只有在准则3下，GC才会把zval收集起来，然后通过新的算法来判断此zval是否为垃圾。

>那么如何判断这么一个变量是否为真正的垃圾呢？

>>简单的说，就是对此zval中的每个元素进行一次refcount减1操作，操作完成之后，如果zval的refcount=0，那么这个zval就是一个垃圾。

>引用手册中的图解:
![PHPCG](/assets/img/201308240103.jpeg)

>A：为了避免每次变量的refcount_gc减少的时候都调用GC的算法进行垃圾判断，此算法会先把所有前面准则3情况下的zval节点放入一个节点(root)缓冲区(root buffer)，并且将这些zval节点标记成紫色，同时算法必须确保每一个zval节点在缓冲区中之出现一次。当缓冲区被节点塞满的时候，GC才开始开始对缓冲区中的zval节点进行垃圾判断。

>B：当缓冲区满了之后，算法以深度优先对每一个zval节点所包含的zval进行减1操作，为了确保不会对同一个zval的refcount重复执行减1操作，一旦zval的refcount减1之后会将zval标记成灰色。

>C：算法再次以深度优先判断每一个节点包含的zval的值，如果zval的refcount等于0，那么将其标记成白色*图中明明为蓝色!*(代表垃圾)，如果zval的refcount大于0，那么将对此zval以及其包含的zval进行refcount加1操作，这个是对非垃圾的还原操作，同时将这些zval的颜色变成黑色（zval的默认颜色属性）.

>D：遍历zval节点，将C中标记成白色的节点zval释放掉。

###PHP中运用新的GC的算法

>在PHP中，GC默认是开启的，你可以通过ini文件中的 zend.enable_gc 项来开启或则关闭GC。当GC开启的时候，垃圾分析算法将在节点缓冲区(roots buffer)满了之后启动。缓冲区默认可以放10,000个节点，当然你也可以通过修改Zend/zend_gc.c中的GC_ROOT_BUFFER_MAX_ENTRIES 来改变这个数值，需要重新编译链接PHP。当GC关闭的时候，垃圾分析算法就不会运行，但是相关节点还会被放入节点缓冲区，这个时候如果缓冲区节点已经放满，那么新的节点就不会被记录下来，这些没有被记录下来的节点就永远也不会被垃圾分析算法分析。如果这些节点中有循环引用，那么有可能产生内存泄漏。之所以在GC关闭的时候还要记录这些节点，是因为简单的记录这些节点比在每次产生节点的时候判断GC是否开启更快，另外GC是可以在脚本运行中开启的，所以记录下这些节点，在代码运行的某个时候如果又开启了GC，这些节点就能被分析算法分析。当然垃圾分析算法是一个比较耗时的操作。

>在PHP代码中我们可以通过gc_enable()和gc_disable()函数来开启和关闭GC，也可以通过调用gc_collect_cycles()在节点缓冲区未满的情况下强制执行垃圾分析算法。这样用户就可以在程序的某些部分关闭或则开启GC，也可强制进行垃圾分析算法。  