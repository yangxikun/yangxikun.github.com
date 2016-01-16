---
layout: post
title: "如何保障用户密码安全"
description: ""
category: WEB安全
tags: []
---
{% include JB/setup %}

#### 如何安全存储密码
- - -
1、使用哈希算法直接对密码进行hash

如`md5(password)`，`sha1(password)`等，这种做法在当前时代已经不安全了，因为随着“彩虹表”不断变大，如果被拖库了，用户的密码就容易被反哈希出来。国内密码学专家[王小云](http://baike.baidu.com/subview/350813/7544439.htm)已成功破解了md5和sha1.

<!--more-->

2、使用哈希算法进行多次hash

如`md5(md5(password))`，其实这种做法存在一定问题：

{% highlight php linenos %}
y = md5(x)
z = md5(y)  //假设是两次hash，数据库存储的是z

//因为哈希算法存在哈希碰撞的问题，那么就会存在
y1 = md5(x1)
y1 = md5(x2) //x1，x2的md5结果都为y1

z = md5(y1)
z = md5(y2) //y1，y2的md5结果都为z

//那么存在
y2 = md5(w1)
y2 = md5(w2) //w1，w2的md5结果都为y2

//所以当password为x1，x2，w1，w2时，都能够通过`md5(md5(password))`得到z！
{% endhighlight %}

3、加盐哈希

* 使用固定的salt：如果有少数几个hash结果被破解了，就容易得出使用的salt了。
* 不同用户使用不同的salt：即使有hash结果被破解了，也很难得出用户的密码。
* salt与hash结果分开存储，每个用户的salt不同，salt与password不是简单的拼接

4、使用更安全的hash函数：SHA256，SHA512，PBKDF2，bcrypt，scrypt，运算消耗资源更多，增加正向破解的难度。

5、使用基于key的哈希函数HMAC(Hash-based Message Authentication Code)，HMAC以密钥和消息为输入，生成一个消息摘要作为输出。这样可以增加反向破解的难度。key应该被存储在一个安全性极高的地方。

安全存储密码的终极目标：即使数据和代码都被拿走了，也没办法破解出密码来。

PHP安全存储密钥：[Safe Password Hashing](http://php.net/manual/en/faq.passwords.php)

#### 正确的姿势传输密码
- - -
网络传输中容易受到各种类型的攻击，最好采用HTTPS协议。


