---
layout: post
title: "php 函数的实现"
description: ""
category: PHP
tags: [PHP底层]
---
{% include JB/setup %}

*学习自[TIPI](http://www.php-internals.com/book/),做个小结,内容从TIPI中选取*

###函数类型

>用户定义的函数

>>在PHP的实现中，如果函数没有显式的返回， Zend引擎也会“帮你“返回NULL。

>内部函数

>>标准函数:count、strpos、implode等

>>语言结构:isset、empty、eval等

>>扩展模块中的函数

>匿名函数

>变量函数

###函数的内部结构

>Zend引擎将函数分为以下类型：

{% highlight cpp linenos %}
#define ZEND_INTERNAL_FUNCTION              1
#define ZEND_USER_FUNCTION                  2  
#define ZEND_OVERLOADED_FUNCTION            3
#define ZEND_EVAL_CODE                      4
#define ZEND_OVERLOADED_FUNCTION_TEMPORARY  5
{% endhighlight %}

>其中的ZEND_USER_FUNCTION是用户函数，ZEND_INTERNAL_FUNCTION是内置的函数。也就是说PHP将内置的函数和 用户定义的函数分别保存。

#####用户函数\(ZEND_USER_FUNCTION\)

>ZE在执行过程中，会将运行时信息存储于_zend_execute_data中：

{% highlight cpp linenos %}
struct _zend_execute_data {
    //...省略部分代码
    zend_function_state function_state;
    zend_function *fbc; /* Function Being Called */
    //...省略部分代码
};
{% endhighlight %}

>在程序初始化的过程中，function_state也会进行初始化，function_state由两个部分组成：

{% highlight cpp linenos %}
typedef struct _zend_function_state {
    zend_function *function;
    void **arguments;
} zend_function_state;
{% endhighlight %}

>\*\*arguments是一个指向函数参数的指针，而函数体本身则存储于\*function中， \*function是一个zend_function结构体， 它最终存储了用户自定义函数的一切信息，它的具体结构是这样的：

{% highlight cpp linenos %}
typedef union _zend_function {
    zend_uchar type;    /* 如用户自定义则为 #define ZEND_USER_FUNCTION 2
                            MUST be the first element of this struct! */
 
    struct {
        zend_uchar type;  /* never used */
        char *function_name;    //函数名称
        zend_class_entry *scope; //函数所在的类作用域
        zend_uint fn_flags;     // 作为方法时的访问类型等，如ZEND_ACC_STATIC等  
        union _zend_function *prototype; //函数原型
        zend_uint num_args;     //参数数目
        zend_uint required_num_args; //需要的参数数目
        zend_arg_info *arg_info;  //参数信息指针
        zend_bool pass_rest_by_reference;
        unsigned char return_reference;  //返回值 
    } common;
 
    zend_op_array op_array;   //函数中的操作
    zend_internal_function internal_function;  
} zend_function;
{% endhighlight %}

>zend_function的结构中的op_array存储了该函数中所有的操作，当函数被调用时，ZE就会将这个op_array中的opline一条条顺次执行， 并将最后的返回值返回。从VLD扩展中查看的关于函数的信息可以看出，函数的定义和执行是分开的，一个函数可以作为一个独立的运行单元而存在。如下所示:

>>Finding entry points
>>Branch analysis from position: 0
>>Return found
>>filename:       /var/www/vld.php
>>function name:  (null)
>>number of ops:  8
>>compiled vars:  none
>>line     \# *  op                           fetch          ext  return  operands
>>---------------------------------------------------------------------------------
>>   2     0  \>   EXT_STMT                                                 
>>         1      NOP                                                      
>>   5     2      EXT_STMT                                                 
>>         3      EXT_FCALL_BEGIN                                          
>>         4      DO_FCALL                                      0          'test'
>>         5      EXT_FCALL_END                                            
>>   7     6      EXT_STMT                                                 
>>         7    \> RETURN                                                   1
>>branch: \#  0; line:     2-    7; sop:     0; eop:     7
>>path \#1: 0, 
>>Function test:
>>Finding entry points
>>Branch analysis from position: 0
>>Return found
>>filename:       /var/www/vld.php
>>function name:  test
>>number of ops:  5
>>compiled vars:  none
>>line     \# *  op                           fetch          ext  return  operands
>>---------------------------------------------------------------------------------
>>   2     0  >   EXT_NOP                                                  
>>   3     1      EXT_STMT                                                 
>>         2      ECHO                                                     'test'
>>   4     3      EXT_STMT                                                 
>>         4    > RETURN                                                   null
>>
>>branch: \#  0; line:     2-    4; sop:     0; eop:     4
>>path \#1: 0, 
>>End of function test.

>注意上面出现两段,第一段的function name为NULL,第二段的function name为test.

#####内部函数\(ZEND_INTERNAL_FUNCTION\)

>ZEND_INTERNAL_FUNCTION函数是由扩展、PHP内核、Zend引擎提供的内部函数，一般用“C/C++”编写，可以直接在用户脚本中调用的函数。如下为内部函数的结构：

{% highlight cpp linenos %}
typedef struct _zend_internal_function {
    /* Common elements */
    zend_uchar type;
    char * function_name;
    zend_class_entry *scope;
    zend_uint fn_flags;
    union _zend_function *prototype;
    zend_uint num_args;
    zend_uint required_num_args;
    zend_arg_info *arg_info;
    zend_bool pass_rest_by_reference;
    unsigned char return_reference;
    /* END of common elements */
 
    void (*handler)(INTERNAL_FUNCTION_PARAMETERS);
    struct _zend_module_entry *module;
} zend_internal_function;
{% endhighlight %}

>最常见的操作是在模块初始化时，ZE会遍历每个载入的扩展模块，然后将模块中function_entry中指明的每一个函数(module->functions)， 创建一个zend_internal_function结构， 并将其type设置为ZEND_INTERNAL_FUNCTION，将这个结构填入全局的函数表(HashTable结构）; 函数设置及注册过程见 Zend/zend_API.c文件中的 zend_register_functions函数。这个函数除了处理函数，也处理类的方法，包括那些魔术方法。

>内部函数的结构与用户自定义的函数结构基本类似，有一些不同，

>>* 调用方法，handler字段. 如果是ZEND_INTERNAL_FUNCTION， 那么ZE就调用zend_execute_internal，通过zend_internal_function.handler来执行这个函数。 而用户自定义的函数需要生成中间代码，然后通过中间代码映射到相对就把方法调用。
>>* 内置函数在结构中多了一个module字段，表示属于哪个模块。不同的扩展其模块不同。
>>* type字段，在用户自定义的函数中，type字段几乎无用，而内置函数中的type字段作为几种内部函数的区分。