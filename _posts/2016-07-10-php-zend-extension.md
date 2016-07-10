---
layout: post
title: "PHP扩展与Zend扩展区别"
description: "PHP扩展与Zend扩展区别"
category: PHP
tags: [PHP扩展]
---
{% include JB/setup %}
**本文参考自PHP Wiki: [Extensions](https://wiki.php.net/internals/extensions#dokuwiki__top)**

#### 数据结构区别
- - -
通常在php.ini中，通过`extension=*`加载的扩展我们称为PHP扩展，通过`zend_extension=*`加载的扩展我们称为Zend扩展，但从源码的角度来讲，PHP扩展应该称为“模块”（源码中以module命名），而Zend扩展称为“扩展”（源码中以extension命名）。

两者最大的区别在于向引擎注册的钩子。少数的扩展，例如xdebug、opcache，既是PHP扩展，也是Zend扩展，但它们在php.ini中的加载方式得用`zend_extension=*`，具体原因下文会说明。

<!--more-->

先来看看两种类型扩展主要的数据结构：

PHP扩展的结构体：

```c
typedef struct _zend_module_entry zend_module_entry;
typedef struct _zend_module_dep zend_module_dep;
 
struct _zend_module_entry {
    unsigned short size;
    unsigned int zend_api;
    unsigned char zend_debug;
    unsigned char zts;
    const struct _zend_ini_entry *ini_entry;
    const struct _zend_module_dep *deps;
    const char *name;
    const struct _zend_function_entry *functions;          /* PHP Functions */
    int (*module_startup_func)(INIT_FUNC_ARGS);             /* MINIT */
    int (*module_shutdown_func)(SHUTDOWN_FUNC_ARGS);        /* MSHUTDOWN */
    int (*request_startup_func)(INIT_FUNC_ARGS);            /* RINIT */
    int (*request_shutdown_func)(SHUTDOWN_FUNC_ARGS);       /* RSHUTDOWN */
    void (*info_func)(ZEND_MODULE_INFO_FUNC_ARGS);
    const char *version;
    size_t globals_size;
#ifdef ZTS
    ts_rsrc_id* globals_id_ptr;
#else
    void* globals_ptr;
#endif
    void (*globals_ctor)(void *global TSRMLS_DC);
    void (*globals_dtor)(void *global TSRMLS_DC);
    int (*post_deactivate_func)(void);
    int module_started;
    unsigned char type;
    void *handle;
    int module_number;
    const char *build_id;
};
 
struct _zend_module_dep {
    const char *name;       /* module name */
    const char *rel;        /* version relationship: NULL (exists), lt|le|eq|ge|gt (to given version) */
    const char *version;    /* version */
    unsigned char type;     /* dependency type */
};
```

Zend扩展的结构体：

```c
/* Typedef's for zend_extension function pointers */
typedef int (*startup_func_t)(zend_extension *extension);
typedef void (*shutdown_func_t)(zend_extension *extension);
typedef void (*activate_func_t)(void);
typedef void (*deactivate_func_t)(void);
 
typedef void (*message_handler_func_t)(int message, void *arg);
 
typedef void (*op_array_handler_func_t)(zend_op_array *op_array);
 
typedef void (*statement_handler_func_t)(zend_op_array *op_array);
typedef void (*fcall_begin_handler_func_t)(zend_op_array *op_array);
typedef void (*fcall_end_handler_func_t)(zend_op_array *op_array);
 
typedef void (*op_array_ctor_func_t)(zend_op_array *op_array);
typedef void (*op_array_dtor_func_t)(zend_op_array *op_array);
 
typedef struct _zend_extension {
    char *name;
    char *version;
    char *author;
    char *URL;
    char *copyright;
 
    startup_func_t startup;
    shutdown_func_t shutdown;
    activate_func_t activate;
    deactivate_func_t deactivate;
 
    message_handler_func_t message_handler;
 
    op_array_handler_func_t op_array_handler;
 
    statement_handler_func_t statement_handler;
    fcall_begin_handler_func_t fcall_begin_handler;
    fcall_end_handler_func_t fcall_end_handler;
 
    op_array_ctor_func_t op_array_ctor;
    op_array_dtor_func_t op_array_dtor;
 
    int (*api_no_check)(int api_no);
    int (*build_id_check)(const char* build_id);
    void *reserved3;
    void *reserved4;
    void *reserved5;
    void *reserved6;
    void *reserved7;
    void *reserved8;
 
    DL_HANDLE handle;
    int resource_number;
} zend_extension;
 
typedef struct _zend_extension_version_info {
    int zend_extension_api_no;
    char *build_id;
} zend_extension_version_info;
```

以xdebug 2.4.0为例，来说明为何其既是PHP扩展，也是Zend扩展。

首先看xdebug.c的2710行处，定义了一个Zend扩展的结构体：

```c
ZEND_DLEXPORT zend_extension zend_extension_entry = {
    XDEBUG_NAME,
    XDEBUG_VERSION,
    XDEBUG_AUTHOR,
    XDEBUG_URL_FAQ,
    XDEBUG_COPYRIGHT_SHORT,
    xdebug_zend_startup,
    xdebug_zend_shutdown,
    NULL,           /* activate_func_t */
    NULL,           /* deactivate_func_t */
    NULL,           /* message_handler_func_t */
    NULL,           /* op_array_handler_func_t */
    xdebug_statement_call, /* statement_handler_func_t */
    NULL,           /* fcall_begin_handler_func_t */
    NULL,           /* fcall_end_handler_func_t */
    xdebug_init_oparray,   /* op_array_ctor_func_t */
    NULL,           /* op_array_dtor_func_t */
    STANDARD_ZEND_EXTENSION_PROPERTIES
};
```

因为xdebug提供了单步调试、性能优化等高级功能，这些是需要hook到Zend引擎才能做到的，而zend_extension这个结构体就提供了hook到Zend引擎的钩子，例如xdebug_statement_call会在每一条PHP语句执行之后调用。

再看xdebug.c的160行处，定义了一个PHP扩展的结构体：

```c
zend_module_entry xdebug_module_entry = {
    STANDARD_MODULE_HEADER,
    "xdebug",
    xdebug_functions,
    PHP_MINIT(xdebug),
    PHP_MSHUTDOWN(xdebug),
    PHP_RINIT(xdebug),
    PHP_RSHUTDOWN(xdebug),
    PHP_MINFO(xdebug),
    XDEBUG_VERSION,
    NO_MODULE_GLOBALS,
    ZEND_MODULE_POST_ZEND_DEACTIVATE_N(xdebug),
    STANDARD_MODULE_PROPERTIES_EX
}
```

xdebug提供了许多函数`xdebug_enable()、xdebug_disable()、xdebug_call_class()……`，这些函数都是定义在xdebug_module_entry中的xdebug_functions。

所以，以我的理解，向用户层面提供一些C实现的PHP函数，需要用到zend_module_entry（即作为PHP扩展），而需要hook到Zend引擎的话，就得用到zend_extension（即作为Zend扩展），xdebug在这里两种都需要。

但是xdebug是通过Zend扩展加载的（`zend_extension=*`），其PHP扩展部分是如何被加载的呢？其实在Zend扩展加载的时候会调用zend_extension中的startup，即xdebug的xdebug_zend_startup函数，在该函数中：

```c
ZEND_DLEXPORT int xdebug_zend_startup(zend_extension *extension)
{
    /* Hook output handlers (header and output writer) */
    xdebug_hook_output_handlers();

    zend_xdebug_initialised = 1;
                                                                                                                                                                                                               
    return zend_startup_module(&xdebug_module_entry);//对xdebug PHP扩展的加载
}
```

#### 加载顺序区别
- - -
分清PHP扩展和Zend扩展的差异后，接着看看扩展是如何加载的，其加载顺序和依赖是怎么处理的：

相关的主要代码片段在main/main.c中：

```c
int php_module_startup(sapi_module_struct *sf, zend_module_entry *additional_modules, uint num_additional_modules)
{
    . . . . . .

    /* this will read in php.ini, set up the configuration parameters,
       load zend extensions and register php function extensions
       to be loaded later */
    if (php_init_config(TSRMLS_C) == FAILURE) {
        return FAILURE;
    }

    . . . . . .

    /* startup extensions statically compiled in */
    if (php_register_internal_extensions_func(TSRMLS_C) == FAILURE) {
        php_printf("Unable to start builtin modules\n");
        return FAILURE;
    }

    /* start additional PHP extensions */
    php_register_extensions_bc(additional_modules, num_additional_modules TSRMLS_CC);

    /* load and startup extensions compiled as shared objects (aka DLLs)
       as requested by php.ini entries
       theese are loaded after initialization of internal extensions
       as extensions *might* rely on things from ext/standard
       which is always an internal extension and to be initialized
       ahead of all other internals
     */
    php_ini_register_extensions(TSRMLS_C);
    zend_startup_modules(TSRMLS_C);

    /* start Zend extensions */
    zend_startup_extensions();

    . . . . . .
}
```

1、php_init_config解析php.ini文件，获取需要加载的PHP扩展和Zend扩展

2、php_register_internal_extensions_func加载静态编译的扩展，静态编译进PHP中的扩展，例如date、ereg、pcre等，这些扩展包含在main/internal_functions.c中的zend_module_entry *php_builtin_extensions[]中

3、php_register_extensions_bc注册SAPI的扩展模块，即additional_modules中的扩展，例如Apache SAPI会注册一些与Apache功能相关的扩展，CLI模式下additional_modules为NULL

4、php_ini_register_extensions中会先加载Zend扩展，之后再加载PHP扩展

```c
void php_ini_register_extensions(TSRMLS_D)
{
    zend_llist_apply(&extension_lists.engine, php_load_zend_extension_cb TSRMLS_CC);//加载Zend扩展
    zend_llist_apply(&extension_lists.functions, php_load_php_extension_cb TSRMLS_CC);//加载PHP扩展

    zend_llist_destroy(&extension_lists.engine);
    zend_llist_destroy(&extension_lists.functions);
}
```

Zend扩展的加载（php_load_zend_extension_cb->zend_load_extension）：

在zend_load_extension中会判断扩展是否合法（能否加载？版本信息是否符合要求？……），如果合法的话将调用zend_register_extension，向全局变量zend_extensions添加当前Zend扩展，并向其他已加载的Zend扩展广播一条消息（实现了message_handler的Zend扩展将接收到）说明自己被加载了，该功能的用处在于，如果某些扩展存在冲突，则扩展能够发出警告信息，并停止加载。

PHP扩展的加载（php_load_php_extension_cb->php_load_extension）：

在php_load_extension中会判断扩展是否合法（能否加载？版本信息是否符合要求？……），如果合法的话将调用zend_register_module_ex，先检查扩展的依赖（_zend_module_entry中的deps），这里只检查当前扩展是否与已加载的扩展冲突，如果没有冲突，则向全局变量module_registry添加当前PHP扩展，接着注册当前PHP扩展的函数（_zend_module_entry中的functions）

这里需要注意的点是deps依赖，该依赖声明了：

* 当前扩展会与哪些扩展冲突
* 哪些扩展需要先于当前扩展加载

但是zend_register_module_ex中只对冲突做检查，如果A声明自己与B冲突，但是B却在A之后加载，那么就无法检查到该冲突了。为了解决这个问题，可以在php.ini中排好PHP扩展的顺序（即B放到A之前），加载的时候会按照该顺序加载。

5、扩展初始化阶段：

先激活PHP扩展，在zend_startup_modules中，会先对PHP扩展进行排序（根据每个PHP扩展中deps声明的依赖），然后执行zend_startup_module_ex，调用PHP扩展的MINIT

再激活Zend扩展，在zend_startup_extensions中，对每个Zend扩展调用其startup()

6、请求初始化阶段：

先调用Zend扩展的activate()：`php_request_startup->zend_activate->init_executor->zend_extension_activator->activate`

再调用PHP扩展的RINIT：`php_request_startup->zend_activate_modules->request_startup_func`

7、请求结束阶段：

先调用PHP扩展的RSHUTDOWN：`php_request_shutdown->zend_deactivate_modules->request_shutdown_func`

再调用Zend扩展的deactivate：`php_request_shutdown->zend_deactivate->shutdown_executor->zend_extension_deactivator->deactivate`

8、扩展关闭阶段：

先调用PHP扩展的MSHUTDOWN：`zend_shutdown->zend_destroy_modules->zend_hash_graceful_reverse_destroy->module_destructor->module_shutdown_func`

再调用Zend扩展的shutdown()：`zend_shutdown->zend_shutdown_extensions->zend_extension_shutdown->shutdown`

#### 一张图了解扩展的生命周期
- - -
图片来自参考文档

![]()
