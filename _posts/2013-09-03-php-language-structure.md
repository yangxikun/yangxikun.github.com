---
layout: post
title: "PHP 语言结构与函数区别"
description: ""
category: PHP
tags: [PHP底层]
---
{% include JB/setup %}

*参考自鸟哥的博文[isset和is_null的不同](http://www.laruence.com/2009/12/09/1180.html)和[TIPI](http://www.php-internals.com/book/)*

###什么是PHP的语言结构?

>即语言本身的一部分,如echo,isset等这些和for,foreach一样,作为PHP语言的组成成分.它们也是PHP的关键字.

###从opcode来看"像函数的"语言结构和函数的区别

>先看如下代码:

{% highlight php linenos %}
<?php
  $a = array('b'=>'c');
    isset( $a['b'] );
    array_key_exists( 'b', $a );
?>
{% endhighlight %}

>输出的opcode如下( php -dvld.active=1 -dvld.execute=1 vld.php )

   2     0  \>   EXT_STMT                                                 
         1      INIT_ARRAY                                       ~0      'c', 'b'
         2      ASSIGN                                                   !0, ~0
   3     3      EXT_STMT                                                 
         4      ZEND_ISSET_ISEMPTY_DIM_OBJ                  2000000  ~2      !0, 'b'
         5      FREE                                                     ~2
   4     6      EXT_STMT                                                 
         7      EXT_FCALL_BEGIN                                          
         8      SEND_VAL                                                 'b'
         9      SEND_VAR                                                 !0
        10      DO_FCALL                                      2          'array_key_exists'
        11      EXT_FCALL_END                                            
   6    12      EXT_STMT                                                 
        13    \> RETURN                                                   1

>从opcode中,可以看出调用array_key_exists函数是需要经过函数调用开始,传递参数,函数执行,函数结束这些步骤,而isset被翻译为ZEND_ISSET_ISEMPTY_DIM_OBJ 指令,isset和array_key_exists可以实现同样的功能,但isset要比array_key_exists快很多,因为少了函数调用所带来的开销.

>语言结构可以有返回值也可以没有,如echo和print,这里的没有和返回null是不同的,任何PHP函数都会有返回值,即使自己定义的函数中没有return,PHP内核也会"帮你"返回NULL.

>所以在能够实现相同功能的情况下,我们应该选择使用语言结构.