---
layout: post
title: "使用PHP进行网页数据抓取小结"
description: ""
category: PHP
tags: [PHP应用]
---
{% include JB/setup %}

#### 抓取思路

1. 通过CURL请求到页面html，假设URL是正确的，这一步需要注意的地方是服务器重定向问题、服务器检查请求的合法性（通常检查请求头，高级的还会通过动态的cookie值例如淘宝指数）。
2. 将请求到的html（所获取的html是未经过js处理的）加载为[DOM](http://www.php.net/manual/zh/book.dom.php)后，使用DOM库提供的方法从网页获取数据，同时也可以结合DOMXPath（获取网页数据XPath的路径可以使用chrome的控制台，在标签上右击鼠标，选择copy xpath，但注意这个xpath可能与你实际想要的有时会不同，注意禁止掉浏览器js运行）。另外一种方法就是使用[php-simple-html-dom](http://www.ecartchina.com/php-simple-html-dom/manual.htm)，这个库提供面向对象的方法像jquery那样从网页获取数据。

<!--more-->

#### 两种获取数据方法比较

DOM是PHP的C扩展，速度上会快很多，但HTML标签位置的变动会导致有时抓取不到数据，所以适合那些页面结构基本不变的网页。

php-simple-html-dom是使用PHP写的一个类库，速度上比较慢，主要是load的时候，对HTML进行解析的时候很慢，但HTML标签位置的变动对其基本没影响，因为它可以根据标签的属性值进行筛选。对于load速度慢的问题，可以将HTML适当的截短到某个标签范围内，来提高速度。

自己实现了个轻量级的类似`simple_html_dom`的类[TagDomRoot](https://github.com/yangxikun/tag-parse)，它的搜索速度会相对快许多，且平均内存占用量较低。