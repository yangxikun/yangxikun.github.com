---
layout: post
title: "转义序列导致的XSS注入"
description: "转义序列导致的XSS注入"
category: WEB安全
tags: [xss]
---
{% include JB/setup %}

#### 问题示例代码
- - -
出现xss注入的视图页面部分代码：

```php
<script>
window.onload = function () {
    var url = '<?php echo htmlspecialchars($_GET['url']); ?>';
    div = document.createElement('div');
    div.innerHTML = '<iframe src="' + url + '"></iframe>';
    document.body.appendChild(div);
}
</script>
```

<!--more-->

如果请求的url是：

```plaintext
http://test.com?url=\42\40\157\156\154\157\141\144\75\42\141\154\145\162\164\50\47\170\163\163\47\51\42\76\74\57\151\146\162\141\155\145\76\74\151\146\162\141\155\145\40\163\162\143\75\42
```

其中，url参数的值为八进制转义序列，转义后为：

```plaintext
" onload="alert('xss')"></iframe><iframe src="
```

那么就会产生一个alert弹框，内容为xss。

在线编码解码：[http://www.jb51.net/tools/zhuanhuan.htm](http://www.jb51.net/tools/zhuanhuan.htm)

#### 分析
- - -
首先查看浏览器渲染出来的html是什么样子的：

![](/assets/img/201611040101.png)

php中输出的是原样的八进制转义序列，在JS中，这样的字符串会被转义（无论是单引号包裹还是双引号包裹着，而PHP对于单引号包裹的字符串不会做转义处理），如下在控制台可以看到转义结果：

![](/assets/img/201611040102.png)

为何php的htmlspecialchars函数没法对`$_GET['url']`进行编码？难道PHP不支持八进制转义序列，从PHP文档中[Escape sequences](http://php.net/manual/en/regexp.reference.escape.php)可知道是支持的，只是转义序列是在字符串字面量（[String literal](https://en.wikipedia.org/wiki/String_literal)）或者正则表达式的模式中才生效。所以`$_GET['url']`的内容不会被PHP进行转义，但是输出到前端时，在JS中，`$_GET['url']`的内容就变为JS的字符串字面量了，会被JS进行转义。

#### 问题解决
- - -
当将PHP输出的内容直接输出为JS代码的一部分时，JS最好做一层encode或者过滤/校验操作，比如解决上面的xss问题代码如下：

```php
<script>
window.onload = function () {
    var url = encodeURI('<?php echo htmlspecialchars($_GET['url']); ?>');
    div = document.createElement('div');
    div.innerHTML = '<iframe src="' + url + '"></iframe>';
    document.body.appendChild(div);
}
</script>
```

浏览器渲染出来的html：

![](/assets/img/201611040103.png)