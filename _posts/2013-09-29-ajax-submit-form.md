---
layout: post
title: "Ajax提交表单"
description: ""
category: PHP
tags: [PHP练手]
---
{% include JB/setup %}

>主要使用了`jQuery ajax - serialize()` 方法.

>The .serialize() method creates a text string in standard URL-encoded notation.

>It can act on a jQuery object that has selected individual form controls, such as `<input>`, `<textarea>`, and `<select>`: `$( "input, textarea, select" ).serialize();`

>html代码:

{% highlight html linenos %}
<html>
    <head>
        <script src="http://code.jquery.com/jquery-1.9.1.js"></script>
        <script>
            $(function(){
                $('#myForm').on('submit',function(e){
                    $.ajax({
                        url:'./ajaxSubmit.php',
                        data:$(this).serialize(),
                        type:'POST',
                        success:function(result){
                            $('#var_dump_POST').append(result);
                        }
                    });
                return false;
            });
            });
        </script>
    </head>
    <body>
        <form action="" method="post" id="myForm">
            <label for="username">username:</label>
            <input type="text" name="username">
            <input type="submit">
        </form>
        <div id="var_dump_POST"></div>
    </body>
</html>
{% endhighlight %}

>php代码:

{% highlight php linenos %}
<?php
    var_dump($_POST);
?>
{% endhighlight %}