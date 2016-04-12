---
layout: post
title: "PHP Generator的实现"
description: "PHP Generator的实现"
category: PHP
tags: []
---
{% include JB/setup %}
对于Generator的实现，关键的数据结构是\_zend\_execute\_data，已在这篇文章：[PHP内核的一点探索——execute_data](http://yangxikun.github.io/php/2016/03/24/php-execute_data.html) 介绍了。本文只是粗略的说明了Generator的实现~

#### Generator的创建
- - -
Generator的实现代码在Zend/zend_generators.c中，我们无法直接new一个Generator对象，如果直接new的话会报如下错误：

`PHP Catchable fatal error:  The "Generator" class is reserved for internal use and cannot be manually instantiated`

在源码中可看到原因：

{% highlight c linenos %}
{% raw %}
static zend_function *zend_generator_get_constructor(zval *object TSRMLS_DC) /* {{{ */
{
    zend_error(E_RECOVERABLE_ERROR, "The \"Generator\" class is reserved for internal use and cannot be manually instantiated");
    
    return NULL;
}
/* }}} */
{% endraw %}
{% endhighlight %}

<!--more-->

PHP手册中也说明了：Generator objects are returned from generators. 也就是需要通过如下方式得到一个Generator对象：

{% highlight php linenos %}
{% raw %}
<?php
function gen() {
    yield;
}
$gen = gen();
var_dump($gen);//Generator对象
{% endraw %}
{% endhighlight %}

那么一个Generator对象是如何生成的？打印上面代码的opcode看看：
![](/assets/img/201604120101.png)

Generator的数据结构：
{% highlight c linenos %}
{% raw %}
typedef struct _zend_generator {
    zend_object std;

    zend_generator_iterator iterator;

    /* The suspended execution context. */
    zend_execute_data *execute_data;

    /* The separate stack used by generator */
    zend_vm_stack stack;

    //yield 能够产生 key => value 形式的值
    /* Current value */
    zval *value;
    /* Current key */
    zval *key;
    /* Variable to put sent value into */
    zval **send_target;//通过send方法传入的变量
    /* Largest used integer key for auto-incrementing keys */
    long largest_used_integer_key;

    /* ZEND_GENERATOR_* flags */
    zend_uchar flags;
} zend_generator;
{% endraw %}
{% endhighlight %}

在初始化gen函数调用时（即上图中的DO\_FCALL），发现gen函数的op_array->fn\_flags有ZEND\_ACC\_GENERATOR标识，说明需要生成一个Generator对象返回。

在Zend/zend_vm_execute.h中：
{% highlight c linenos %}
{% raw %}
            ……
    if (UNEXPECTED((EG(active_op_array)->fn_flags & ZEND_ACC_GENERATOR) != 0)) {
        if (RETURN_VALUE_USED(opline)) {
            ret->var.ptr = zend_generator_create_zval(EG(active_op_array) TSRMLS_CC);//创建Generator对象
            ret->var.fcall_returned_reference = 0;
        }
    }
            ……
{% endraw %}
{% endhighlight %}

在Zend/zend_generators.c中，zend_generator_create_zval函数主要做的事情就是备份当前EG中的一些与执行上下文相关的信息，然后创建新的execute_data，同时也分配新的一份\_zend\_vm\_stack，注意该堆栈与原本的EG(argument_stack)是分开的，也即每个Generator对象都会拥有自己的\_zend\_vm\_stack：
{% highlight c linenos %}
{% raw %}
ZEND_API zval *zend_generator_create_zval(zend_op_array *op_array TSRMLS_DC) /* {{{ */
{
            ……//备份EG中的一些与执行上下文相关信息
    execute_data = zend_create_execute_data_from_op_array(op_array, 0 TSRMLS_CC);
            ……//恢复EG中的一些与执行上下文相关信息，使得原来的上下文环境可以继续正常执行
    //创建一个Generator对象后返回
}
{% endraw %}
{% endhighlight %}

在zend_create_execute_data_from_op_array中实际调用的是Zend/zend_execute.c中的：
{% highlight c linenos %}
{% raw %}
static zend_always_inline zend_execute_data *i_create_execute_data_from_op_array(zend_op_array *op_array, zend_bool nested TSRMLS_DC) /* {{{ */
{
            ……
    if (UNEXPECTED((op_array->fn_flags & ZEND_ACC_GENERATOR) != 0)) {
            ……
        EG(argument_stack) = zend_vm_stack_new_page((total_size + (sizeof(void*) - 1)) / sizeof(void*));//新的堆栈，与先前的堆栈是分离的
        EG(argument_stack)->prev = NULL;//这里其实不需要，zend_vm_stack_new_page已经做了= =
        execute_data = (zend_execute_data*)((char*)ZEND_VM_STACK_ELEMETS(EG(argument_stack)) + args_size + execute_data_size + Ts_size);
        //之所以这里需要生成一个prev_execute_data，是因为要能够传递函数参数进来，这就需要了解函数参数是如何传递的，下文会提到
        //在上文的例子中gen函数并没有参数
        /* copy prev_execute_data */
        EX(prev_execute_data) = (zend_execute_data*)((char*)ZEND_VM_STACK_ELEMETS(EG(argument_stack)) + args_size);                                                                                          
        memset(EX(prev_execute_data), 0, sizeof(zend_execute_data));
        EX(prev_execute_data)->function_state.function = (zend_function*)op_array;
        EX(prev_execute_data)->function_state.arguments = (void**)((char*)ZEND_VM_STACK_ELEMETS(EG(argument_stack)) + ZEND_MM_ALIGNED_SIZE(sizeof(zval*)) * args_count);

        /* copy arguments */
        *EX(prev_execute_data)->function_state.arguments = (void*)(zend_uintptr_t)args_count;
        if (args_count > 0) {
            zval **arg_src = (zval**)zend_vm_stack_get_arg_ex(EG(current_execute_data), 1);
            zval **arg_dst = (zval**)zend_vm_stack_get_arg_ex(EX(prev_execute_data), 1);
            int i;

            for (i = 0; i < args_count; i++) {
                arg_dst[i] = arg_src[i];
                Z_ADDREF_P(arg_dst[i]);
            }
        }
    } else {
            ……
    }
            ……
}
{% endraw %}
{% endhighlight %}

上面新建好的上下文执行堆栈如下图：
![](/assets/img/201604120102.png)

#### Generator的执行
- - -
通过调用send、next、throw方法能够使Generator执行到下一个yield或结束。

在Generator的方法的实现中，基本都会调用如下一个函数：

{% highlight c linenos %}
{% raw %}
static void zend_generator_ensure_initialized(zend_generator *generator TSRMLS_DC) /* {{{ */
{
    if (generator->execute_data && !generator->value) {     //如果还未执行过
        zend_generator_resume(generator TSRMLS_CC);         //首次执行
        generator->flags |= ZEND_GENERATOR_AT_FIRST_YIELD;  //设置标识，表明现在已经执行到第一个yield处了
    }
}
/* }}} */
{% endraw %}
{% endhighlight %}

在Zend/zend_generators.c中，以send方法为例：
{% highlight c linenos %}
{% raw %}
/* {{{ proto mixed Generator::send(mixed $value)
 * Sends a value to the generator */
ZEND_METHOD(Generator, send)
{
    zval *value;
    zend_generator *generator;

    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "z", &value) == FAILURE) {//接收传递过来的参数
        return;
    }

    generator = (zend_generator *) zend_object_store_get_object(getThis() TSRMLS_CC);//获取当前对象对应的generator

    zend_generator_ensure_initialized(generator TSRMLS_CC);//确保首次执行过了

    /* The generator is already closed, thus can't send anything */
    if (!generator->execute_data) {
        return;
    }

    /* Put sent value in the target VAR slot, if it is used */
    if (generator->send_target) {
        Z_DELREF_PP(generator->send_target);
        Z_ADDREF_P(value);
        *generator->send_target = value;
    }

    zend_generator_resume(generator TSRMLS_CC);//执行一次

    if (generator->value) {//如果有返回值
        RETURN_ZVAL_FAST(generator->value);
    }
}
/* }}} */
{% endraw %}
{% endhighlight %}

Generator的执行关键在zend_generator_resume函数中：
{% highlight c linenos %}
{% raw %}
ZEND_API void zend_generator_resume(zend_generator *generator TSRMLS_DC) /* {{{ */
{
    //这个函数中的注释已经很详细了~
            ……
    /* Backup executor globals 备份EG中与上下文相关的信息 */
    zval **original_return_value_ptr_ptr = EG(return_value_ptr_ptr);
    zend_execute_data *original_execute_data = EG(current_execute_data);
    zend_op **original_opline_ptr = EG(opline_ptr);
    zend_op_array *original_active_op_array = EG(active_op_array);
    HashTable *original_active_symbol_table = EG(active_symbol_table);
    zval *original_This = EG(This);
    zend_class_entry *original_scope = EG(scope);
    zend_class_entry *original_called_scope = EG(called_scope);
    zend_vm_stack original_stack = EG(argument_stack);
    
    /* We (mis)use the return_value_ptr_ptr to provide the generator object
     * to the executor, so YIELD will be able to set the yielded value */
    EG(return_value_ptr_ptr) = (zval **) generator;//作为对generator的引用，为了能够在执行yield opcode时设置generator->value
    
    /* Set executor globals 恢复Generator的执行上下文信息 */
    EG(current_execute_data) = generator->execute_data;
    EG(opline_ptr) = &generator->execute_data->opline;
    EG(active_op_array) = generator->execute_data->op_array;
    EG(active_symbol_table) = generator->execute_data->symbol_table;
    EG(This) = generator->execute_data->current_this;
    EG(scope) = generator->execute_data->current_scope;
    EG(called_scope) = generator->execute_data->current_called_scope;
    EG(argument_stack) = generator->stack;
    
    //这里解释了为何在创建generator的execute_data时生成一个prev_execute_data的原因
    /* We want the backtrace to look as if the generator function was
     * called from whatever method we are current running (e.g. next()).
     * The first prev_execute_data contains an additional stack frame,
     * which makes the generator function show up in the backtrace and
     * makes the arguments available to func_get_args(). So we have to
     * set the prev_execute_data of that prev_execute_data :) */
    generator->execute_data->prev_execute_data->prev_execute_data = original_execute_data;
    
    /* Resume execution 执行Generator，从上次停止的地方继续执行opcode*/
    generator->flags |= ZEND_GENERATOR_CURRENTLY_RUNNING;
    zend_execute_ex(generator->execute_data TSRMLS_CC);
    generator->flags &= ~ZEND_GENERATOR_CURRENTLY_RUNNING;

    /* Restore executor globals 恢复原来调用Generator执行的上下文*/
    EG(return_value_ptr_ptr) = original_return_value_ptr_ptr;
    EG(current_execute_data) = original_execute_data;
    EG(opline_ptr) = original_opline_ptr;
    EG(active_op_array) = original_active_op_array;
    EG(active_symbol_table) = original_active_symbol_table;
    EG(This) = original_This;
    EG(scope) = original_scope;
    EG(called_scope) = original_called_scope;
    EG(argument_stack) = original_stack;

    /* If an exception was thrown in the generator we have to internally
     * rethrow it in the parent scope. */
    if (UNEXPECTED(EG(exception) != NULL)) {
        zend_throw_exception_internal(NULL TSRMLS_CC);
    }
}
/* }}} */
{% endraw %}
{% endhighlight %}

#### Generator的结束
- - -
每个Generator的op_array中都会有GENERATOR_RETURN opcode，不是普通函数的RETURN，看看其handler：
{% highlight c linenos %}
{% raw %}
static int ZEND_FASTCALL  ZEND_GENERATOR_RETURN_SPEC_HANDLER(ZEND_OPCODE_HANDLER_ARGS)
{
    /* The generator object is stored in return_value_ptr_ptr */
    zend_generator *generator = (zend_generator *) EG(return_value_ptr_ptr);

    /* Close the generator to free up resources */
    zend_generator_close(generator, 1 TSRMLS_CC);//当前generator已经执行完

    /* Pass execution back to handling code */
    ZEND_VM_RETURN();//返回到zend_generator_resume中
}
{% endraw %}
{% endhighlight %}

#### 关于php中用户自定义函数参数的传递
- - -
带有参数的函数，其op\_array中存在一个RECV opcode。因为函数的参数值是存在于其调用域的execute_data中的，在函数对应的execute_data中其实没有。

查看RECV的handler函数：
{% highlight c linenos %}
{% raw %}
static int ZEND_FASTCALL  ZEND_RECV_INIT_SPEC_CONST_HANDLER(ZEND_OPCODE_HANDLER_ARGS)
{
            ……
    zval **param = zend_vm_stack_get_arg(arg_num TSRMLS_CC);
            ……
}

static zend_always_inline int zend_vm_stack_get_args_count(TSRMLS_D)
{
    return zend_vm_stack_get_args_count_ex(EG(current_execute_data)->prev_execute_data);
}

static zend_always_inline zval** zend_vm_stack_get_arg(int requested_arg TSRMLS_DC)
{
    //从上一个execute_data中获取参数的值
    return zend_vm_stack_get_arg_ex(EG(current_execute_data)->prev_execute_data, requested_arg);
}
{% endraw %}
{% endhighlight %}