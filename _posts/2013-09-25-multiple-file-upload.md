---
layout: post
title: "多个文件上传"
description: ""
category: PHP
tags: [PHP练手]
---
{% include JB/setup %}

>html代码如下,有两点需要注意的:一是设置form的enctype属性,二是使用post方法

{% highlight html linenos %}
<html>
<head>
    <meta charset="utf-8">
</head>
<body>
    <form action="test.php" method="post" enctype="multipart/form-data">
        <input type="file" name="userfile1">
        <input type="file" name="userfile2">
        <input type="file" name="userfile3">
        <input type="submit">
    </form>
</body>
</html>
{% endhighlight %}

>php代码如下:

{% highlight php linenos %}
<?php
/**
 * 错误代码:
 * 
 * UPLOAD_ERR_OK
 *
 *    其值为 0，没有错误发生，文件上传成功。
 * UPLOAD_ERR_INI_SIZE
 *
 *    其值为 1，上传的文件超过了 php.ini 中 upload_max_filesize 选项限制的值。
 * UPLOAD_ERR_FORM_SIZE
 *
 *    其值为 2，上传文件的大小超过了 HTML 表单中 MAX_FILE_SIZE 选项指定的值。
 * UPLOAD_ERR_PARTIAL
 *
 *    其值为 3，文件只有部分被上传。
 * UPLOAD_ERR_NO_FILE
 *
 *    其值为 4，没有文件被上传。
 * UPLOAD_ERR_NO_TMP_DIR
 *
 *    其值为 6，找不到临时文件夹。PHP 4.3.10 和 PHP 5.0.3 引进。
 * UPLOAD_ERR_CANT_WRITE
 *
 *    其值为 7，文件写入失败。PHP 5.1.0 引进。
 *
 */
if ( !empty( $_FILES ) ) {
    foreach ( $_FILES as $key => $value ) {
        if ( $value['error'] === 0 ) {//上传过程中未出错
            if ( is_uploaded_file( $value['tmp_name'] ) ) {
                if ( move_uploaded_file( $value['tmp_name'], /*...目的位置*/ ) ) {
                    echo 'file '.$value['name'].' has been uploaded.<br />';
                } else {
                    echo 'Cann\'t upload file!Maybe permission deny!';
                }
            }
        }
    }
}
?>
{% endhighlight %}