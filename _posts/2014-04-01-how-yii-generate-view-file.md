---
layout: post
title: "Yii是如何产生视图文件？"
description: ""
category: PHP
tags: [PHP应用]
---
{% include JB/setup %}
一般我们会在控制器中调用：`$this->render('contact', array('model'=>$model));`。

在经过一系列预处理后，会调用`CBaseController`的方法：

{% highlight c linenos %}
public function renderInternal($_viewFile_,$_data_=null,$_return_=false)
{
    // we use special variable names here to avoid conflict when extracting data
    if(is_array($_data_))
        extract($_data_,EXTR_PREFIX_SAME,'data');
    else
        $data=$_data_;
    if($_return_)
    {
        ob_start();
        ob_implicit_flush(false);
        require($_viewFile_);
        return ob_get_clean();
    }
    else
        require($_viewFile_);
}
{% endhighlight %}

* 该方法先判断`render()`的第二个参数是否为数组，如果为数组，导出变量，否则直接赋值给`$data`。
* 如果需要返回，那么会通过PHP Output Control功能将要输出的视图文件转换为字符串返回。
* 否则直接包含文件。

<!--more-->
对于PHP的输出内容，一般来说它会经过`PHP缓冲->Apache缓冲->内核缓冲->客户端缓冲->显示`这几个地方。

* `ob_start()`打开输出缓冲，现在PHP默认打开了输出缓冲。
* `ob_implicit_flush(false)`禁止PHP自动送出缓冲区内容，即禁止自动执行`flush()`。
* 加载视图文件
* `ob_get_clean()`返回缓冲区内容，并清空。

与输出缓冲相关的`php.ini`配置：


1. `output_buffering`   值可以为on、off、整数，默认为4096。
2. `output_handler`  默认为 null，其值只能设置为一个内置的函数名，作用就是将脚本的所有输出，用所定义的函数进行处理。他的用法和`ob_start('function_name')`较类似，配置文件中建议使用`zlib.output_handler`代替该指令。
3. `implicit_flush` 默认为off，当设置为on时，PHP将在输出后，自动送出缓冲区内容，即自动执行`flush()`。