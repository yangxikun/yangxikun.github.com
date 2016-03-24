---
layout: post
title: "PHP execute_data"
description: "php execute_data"
category: PHP
tags: []
---
{% include JB/setup %}
这阵子想研究下PHP Generator的实现，发现对于Generator很重要的一个数据结构为`_zend_execute_data`，在PHP源码中通常是一个`execute_data`变量，与该变量相关的宏是`#define EX(element) execute_data.element`。

查看了下opcode的执行，发现`_zend_execute_data`对于PHP代码的执行也是关键的数据结构。所以本文主要围绕该数据结构展开来讲~

<!--more-->
#### PHP代码的执行
- - -
总所周知，PHP代码的执行会经历：词法分析->语法分析->opcode生成->执行opcode。

对“opcode生成->执行opcode”展开来就是：

1. `zend_compile_file()`将PHP代码文件编译为op_array（opcode的数组）
1. 初始化一个execute_data，之后赋值：`EX(op_array) = op_array;EX(opline) = op_array->opcodes;`
1. 执行opcode：`EX(opline)->handler(execute_data)`，这是在一个while语句中执行的。

execute_data可以理解为一个执行上下文，通过gdb单步调试，发现：

当我们执行`php hello.php`时，会创建一个execute_data，当出现include/require或函数调用时，则会新建一个execute_data后，切换到新的execute_data执行，当一个execute_data执行结束，会切换到上层的execute_data继续执行。

![](/assets/img/201603240101.png)

每一个op_array的最后一条opcode都是RETURN，这也是为什么在类似PHP框架的配置文件中可以直接`<?php return [配置信息]; ?>`，而加载配置文件的代码为`$config = require('配置文件');`。

通过VLD扩展可以打印出一个PHP执行文件产生的opcode：

![](/assets/img/201603240102.png)

如上图，就存在两个op_array，一个是FunCall.php文件的，一个是foo函数的。

#### \_zend\_execute\_data结构体说明
- - -
*未标注释的结构体成员尚未了解其作用*
![](/assets/img/201603240103.png)

以一个例子说明下EX(object)、EX(current_scope)、EX(current_called_scope)、EX(current_this)、EX(call)、EG(This)、EG(scope)、EG(called_scope)在函数调用时的变化：

例子代码：
{% highlight php linenos %}
{% raw %}
<?php
class bar
{
    static public function pp($param)
    {
        echo $param;
    }
}
class foo
{
    public function p($param)
    {
        self::spp($param);
        bar::pp($param);
    }

    static private function spp($param)
    {
        echo $param;
    }
}

function ff()
{
    $foo = new foo();
    $foo->p("haha\n");
}
ff();

{% endraw %}
{% endhighlight %}
调用图（图太大，请另起一个标签页查看= =）：
![](/assets/img/201603240104.png)

#### \_zend\_execute\_data结构体的初始化
- - -
在Zend/zend_execute.c中的函数`zend_execute_data *i_create_execute_data_from_op_array(zend_op_array *op_array, zend_bool nested TSRMLS_DC)`负责创建新的execute_data，其内存分配是在EG(argument_stack)（指向某一段内存，\_zend\_vm\_stack数据结构，不足时会增长）上进行的。

{% highlight c linenos %}
{% raw %}
typedef struct _zend_vm_stack *zend_vm_stack;
struct _zend_vm_stack {
    void **top;
    void **end;
    zend_vm_stack prev;
};
{% endraw %}
{% endhighlight %}

在一个初始化好的EG(argument_stack)上分配一个execute_data：
![](/assets/img/201603240105.png)

EG(argument_stack)上没有足够的内存分配一个execute_data时，新申请一个\_zend\_vm\_stack：
![](/assets/img/201603240106.png)