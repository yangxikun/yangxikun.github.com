---
layout: post
title: "nginx配置注意的问题"
description: ""
category: nginx
tags: [nginx配置]
---
{% include JB/setup %}

#### $document_root
- - -

    这个全局变量是由root配置的，当有一点需要注意的就是，这个root是以php-fpm的worker进程的chroot目录作为根目录的。

    假如网站根目录：`/var/www`；请求`http://localhost/index.php`，php-fpm配置worker的chroot为：`/var/www`，那么nginx.conf中的root就应该设置为`/`。
