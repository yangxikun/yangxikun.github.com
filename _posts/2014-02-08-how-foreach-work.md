---
layout: post
title: "PHP foreach 是如何遍历数组的？"
description: ""
category: PHP
tags: [PHP底层]
---
{% include JB/setup %}
*参考资料：[How foreach actually works](http://stackoverflow.com/questions/10057671/how-foreach-actually-works/14854568#14854568)和[PHP的哈希表实现](http://www.php-internals.com/book/?p=chapt03/03-01-02-hashtable-in-php)*

#### Array类型的实现
- - -
在PHP的zvalue_value结构体中，我们知道array类型是通过HashTable实现的，结构如下所示：
{% highlight cpp linenos %}
typedef struct _hashtable { 
    uint nTableSize;        // hash Bucket的大小，最小为8，以2x增长。
    uint nTableMask;        // nTableSize-1 ， 索引取值的优化
    uint nNumOfElements;    // hash Bucket中当前存在的元素个数，count()函数会直接返回此值 
    ulong nNextFreeElement; // 下一个数字索引的位置
    Bucket *pInternalPointer;   // 当前遍历的指针（foreach比for快的原因之一）
    Bucket *pListHead;          // 存储数组头元素指针
    Bucket *pListTail;          // 存储数组尾元素指针
    Bucket **arBuckets;         // 存储hash数组
    dtor_func_t pDestructor;    // 在删除元素时执行的回调函数，用于资源的释放
    zend_bool persistent;       //指出了Bucket内存分配的方式。如果persisient为TRUE，则使用操作系统本身的内存分配函数为Bucket分配内存，否则使用PHP的内存分配函数。
    unsigned char nApplyCount; // 标记当前hash Bucket被递归访问的次数（防止多次递归）
    zend_bool bApplyProtection;// 标记当前hash桶允许不允许多次访问，不允许时，最多只能递归3次
#if ZEND_DEBUG
    int inconsistent;
#endif
} HashTable;
{% endhighlight %}

`Bucket *pInternalPointer;`就是`foreach`用于遍历`Bucket`的指针，数组的每一个元素都存储在`Bucket`，它比`for`快，因为`for`需要对`key`进行哈希后，才能找到相应节点。

<!--more-->

#### 请看下面的示例代码以了解foreach对数组的遍历
- - -
Test Case 1:
{% highlight php linenos %}
<?php
$arr = [1, 2, 3, 4, 5];

foreach ($arr as $key => $value) {
    echo "$value\n";
}
?>
{% endhighlight %}

上面的代码很简单，`foreach`使用`pInternalPointer`逐个遍历`Bucket`，输出结果可想而知：<br />
![test1](/assets/img/201402080101.png)

Test Case 2:
{% highlight php linenos %}
<?php
$arr = [1, 2, 3, 4, 5];
var_dump(each($arr));//先将pInternalPointer往前挪一位
foreach ($arr as $key => $value) {
    echo "$value\n";
}
var_dump(current($arr));//输出当前pInternalPointer指向的Bucket
?>
{% endhighlight %}

根据[Manual](http://php.net/manual/en/control-structures.foreach.php)所说：

When foreach first starts executing, the internal array pointer is automatically reset to the first element of the array. This means that you do not need to call reset() before a foreach loop.

那么示例代码中的`foreach`应该能正常输出1-5了：<br />
![test2](/assets/img/201402080102.png)

从输出结果图中，可以看到最后的`var_dump(current($arr));`输出结果为false，原因就在于`foreach`发现`$arr`的`refcount__gc`为1，`is_ref__gc`为0，此时`foreach`并不会复制`$arr`指向的zval，而是将`refcount__gc`的值加1。当foreach在遍历元素时，使用的就是$arr的`(zval).value->ht->pInternalPointer`，所以当遍历结束时，`pInternalPointer`已经指向null了，因此`var_dump(current($arr));`的输出结果为false。

Test Case 3:
{% highlight php linenos %}
<?php
$arr = [1, 2, 3, 4, 5];
$t = $arr;
foreach ($arr as $key => $value) {
    echo "$value\n";
}
var_dump(current($arr));
?>
{% endhighlight %}

输出结果：<br />
![test3](/assets/img/201402080103.png)

对于TC3的示例代码，`var_dump(current($arr));`输出了1。为什么不是false呢？因为foreach发现`$arr`的`refcount__gc`为2，`is_ref__gc`为0，也就是说有变量“引用”了`$arr`，如果改变了`$arr`的`pInternalPointer`，那么`$t`的`pInternalPointer`也会被改变，为了不影响`$t`，`foreach`对`$arr`指向的zval进行了复制之后再遍历。

Test Case 4:
{% highlight php linenos %}
<?php
$arr = [1, 2, 3, 4, 5];
$t = &$arr;
foreach ($arr as $key => $value) {
    echo "$value\n";
}
var_dump(current($arr));
?>
{% endhighlight %}

在TC3的基础上，将赋值改为引用，看看输出结果：<br />
![test3](/assets/img/201402080104.png)

`var_dump(current($arr));`的输出又变回false了。这是因为foreach发现`$arr`的`is_ref__gc`为1，说明有其它变量引用了`$arr`，这时，foreach就会将其和TC2的情况同等看待，并不会复制`$arr`的zval。

Test Case 5:
{% highlight php linenos %}
<?php
$arr = [1, 2, 3, 4, 5];
foreach ($arr as $key => &$value) {
    echo "$value\n";
}
var_dump(current($arr));
?>
{% endhighlight %}

TC5的输出结果将和TC4一样，根据[Manual](http://php.net/manual/en/control-structures.foreach.php)所说：

In order to be able to directly modify array elements within the loop precede $value with &.

所以这里的`$arr`的`is_ref__gc`为1，没有发生zval复制。

这样就结束了吗？还没有呢，继续看

Test Case 6:
{% highlight php linenos %}
<?php
$arr = [1, 2, 3, 4, 5];
foreach ($arr as $key => $value) {
    echo "$value\n";
    var_dump(current($arr));
}
?>
{% endhighlight %}

输出结果：<br />
![test6](/assets/img/201402080105.png)

为什么`var_dump(current($arr));`的输出都是2呢？`foreach`应该没有复制`$arr`呀？

首先解释为什么都是2，因为foreach其实是这样运行的，看下面伪代码：
{% highlight php linenos %}
<?php
reset();
while (get_current_data(&data) == SUCCESS) {
    move_forward();
    code();//var_dump(current($arr));在这里执行
}
?>
{% endhighlight %}
1、取值2、指针往前移3、执行用户代码。

对于第二个问题，`foreach`并没有复制`$arr`，关键在于`current()`函数的调用，该函数是通过引用传递的，`current()`发现`$arr`的`refcount__gc`为2，`is_ref__gc`为0，所以就进行了zval的分离，复制了一份zval，`$arr`就指向了这分新的zval，而foreach仍然使用“旧”的zval在遍历。

Test Case 7:
{% highlight php linenos %}
<?php
$arr = [1, 2, 3, 4, 5];
$obj = (object) [6, 7, 8, 9, 10];

$ref =& $arr;
foreach ($ref as $val) {
    echo "$val\n";
    if ($val == 3) {
        $ref = $obj;
    }
}
?>
{% endhighlight %}

如果已经理解了上面6个示例，那么TC7的输出结果你也应该能分析出来了：<br />
![test7](/assets/img/201402080106.png)

#### 总结
- - -
如果`foreach`要遍历一个数组：
1. 数组的`refcount__gc`为1，`is_ref__gc`为0，那么`foreach`并不会复制zval；
2. 数组的`refcount__gc`>1，`is_ref__gc`为0，那么`foreach`将会复制zval；
3. 数组的`is_ref__gc`为1，那么`foreach`并不会复制zval；
4. 注意在遍历的时候也会发生数组zval的复制，如TC6。

#### 附加的一个问题
- - -
有人问道当foreach发生zval复制时，从上面的例子可以得出这样的结论：`(zval).value->ht`会被复制一份，那么`(zval).value->ht->arBuckets`即该二级指针存储的`Bucket`是否也会被复制？

看这段示例代码：
{% highlight php linenos %}
<?php
$arr = [1, 2, 3, 4, 5];

$ref = $arr;
foreach ($arr as $val) {
    echo "$val\n";
    $arr[] = $val + 1;
}
var_dump($arr);
?>
{% endhighlight %}

如果`(zval).value->ht->arBuckets`没有被复制，那么`foreach`的输出就不止1、2、3、4、5了，可结果如下：
![test8](/assets/img/201402080107.png)

从输出结果可以看出，`(zval).value->ht->arBuckets`也是会被复制的。