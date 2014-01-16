---
layout: post
title: "PHP安全最佳实践"
description: ""
category: PHP
tags: [PHP安全]
---
{% include JB/setup %}
翻译自：[PHP Security Best Practices](http://anchetawern.github.io/blog/2013/12/15/php-security-best-practices/)

In this post were going to have a look at some of the best practices in PHP when it comes to security.

在这一篇博文中将会探讨一些关于PHP安全的最佳实践。

#### Always Update
#### 经常更新

If possible always use the latest stable release of PHP because it contains some security updates and bug fixes. This will make applications written on top of it more secure.

如果可能，使用最新发布的稳定版PHP。因为它包含了一些安全的更新和BUG的修复。这能够让PHP应用更加安全。

<!--more-->
#### Secure Configuration
#### 安全配置

* Disable exposure of which PHP version your server is using. You can do it by searching for expose_php in your php.ini file and set it to Off:
* 在php.ini中，设置`expose_php = Off`。

This will disable the inclusion of the PHP version in the response headers under the `X-Powered-By` attribute. Here’s an example of a site which has set `expose_php` to `On`. As you can see the value `X-Powered-By` attribute is `PHP/5.4.17` so we pretty much know which PHP version the server is running. An attacker can use this information to exploit the security vulnerabilities of this specific PHP version.

这会使PHP的版本信息在HTTP响应头中`X-Powered-By`不会出现。一个攻击者可以利用这个信息进行漏洞攻击。（PS：在HTTP响应头中的Server字段也会暴露服务器操作系统信息和WEB服务器信息，Apache的话需要在编译的时候进行一些修改，才能改变这个字段的值）。

* Make sure that you don’t have any files in your server that calls the `phpinfo()` function. If you want to make use of it, make sure the filename can’t easily be guessed like `phpinfo.php` and don’t store it on the root of your web accessible directory. Don’t forget to delete it once you’re done.
* 谨慎的使用`phpinfo()`函数，不要被别人访问到！（PS：phpMyadmin也一样。）

* Log errors instead of displaying them. Errors, notices and warnings in your web application can provide valuable information to attackers such as filenames and the name of fields that you used on your tables. Make sure you set the following in your php.ini file:
{% highlight php linenos %}
display_startup_errors = Off #disable displaying of startup errors
display_errors = Off #disable displaying of errors
html_errors = Off #disable formatting of errors in HTML
error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT #report all errors, warnings and notices including coding standards
log_errors = On #log errors to a file
{% endhighlight %}
* 记录错误，而不是暴露错误。像上面那样配置你的php.ini。

* Disable file uploads when not needed.
* 当你不需要上传文件时，关闭文件上传功能`file_uploads = Off
`！

If your web application has a file upload feature then you need to make sure that you know some of the best practices in securing file uploads. Here’s a good article from Sitepoint on how to create a secure file upload in PHP. You can also make use of a library that’s specifically created for handling file uploads such as the Upload library from Josh Lockhart(Codeguy).

如果你的应用需要上传文件，你必须确保文件上传的安全性。这里有一篇很好的博文[关于如何使用PHP创建一个安全的文件上传](http://www.sitepoint.com/file-uploads-with-php/)。你也可以使用这个[库](https://github.com/codeguy/Upload)来处理文件上传。

* Disable remote file execution. If you don’t need to use functions such as `fopen`, `fsockopen` or `file_get_contents` then you can just set `allow_url_fopen` to `Off`. Curl can provide with similar functionality so most of the time you won’t really need it.
* 关闭远程文件操作。设置`allow_url_fopen = Off`和`allow_url_include = Off`。

* Limit the maximum size of POST data to a value that you think is enough for your web application needs. This is to prevent attackers from flooding your web application by POSTing huge amount of data. Note that this can be expressed in kilo (K), mega (M) or giga (G).
* 限制POST数据的大小为你的应用所需要的。因为这可以防止攻击者对你发动洪流攻击通过发送数据量很大的POST请求（这会导致服务器带宽被大量占用）。注意，这个值可以设置为K、M或G。

* Do note that the value that you set for `post_max_size` should be larger than the `upload_max_filesize` since uploaded files are also submitted via POST.
* 注意`post_max_size`的值要比`upload_max_filesize`大，因为POST的数据不仅包含了上传的文件，还有头信息等。

* `memory_limit` should also be larger than the `post_max_size`.
* `memory_limit`也应该要比`post_max_size`大。

* Limit maximum input time. This will limit the amount of time for PHP to parse input data from either `$_POST` or `$_GET`. Note that the value is expressed in seconds.
* 限制input时间。这会限制PHP对`$_POST` 或 `$_GET`的解析时间。

* Limit maximum execution time to a reasonable value. The default value of 30 seconds seems reasonable enough.
* 限制最大执行时间，这个是指使用的CPU秒数。

* Limit the use of shell functions such as `exec`, `passthru`, `shell_exec`, `proc_open`, and `popen`. If there’s no other option for implementing something and you absolutely need to use it make sure that users of your web application will not be able to execute any system commands. If you need user input for executing system commands then make sure that you’re validating the data correctly.
* 限制对一些涉及到系统安全函数的使用，例如`exec`, `passthru`, `shell_exec`, `proc_open`, and `popen`。

* Only allow execution of PHP files on a specific directory.
* 仅仅允许PHP文件在特定目录执行`open_basedir = /var/www/public_html`。

* Set temporary upload directory to a path outside of the `open_base_dir`. This prevents files in the temporary upload directory from being executed.
* 设置`upload_tmp_dir`与`open_base_dir`不同。这样可以避免上传的文件被执行。

* Make sure that your web accessible directory is set to read-only.
* 确保你的web目录只能被读。

#### Use CURL
#### 使用CURL

Always use the CURL extension when making requests to other servers especially if you’re working with sensitive data. This is because CURL is by default makes requests securely over SSL/TLS (Secure Socket Layer/Transport Security Layer). Here’s an example of how to perform requests using CURL:
{% highlight php linenos %}
<?php
$url = 'https://bitpay.com/api/invoice';
$req = curl_init($url);
curl_setopt($req, CURLOPT_RETURNTRANSFER, TRUE);
$response = curl_exec($req);
?>
{% endhighlight %}
使用CURL当需要向其他服务器发送请求的时候，特别是含有敏感数据。因为CURL可以发送HTTPS的请求。

Also make sure to set the following options when you’re working with sensitive data:
确保一下的选项被设置，当你传输敏感数据的时候：

> * `CURLOPT_SSL_VERIFYPEER` – should be set to `TRUE` always. This will tell CURL to check if the remote certificate of the server where you’re performing a request is valid.
> * `CURLOPT_SSL_VERIFYHOST` – should be set to `TRUE` always. This tells CURL to check that the Certificate was issued to the entity that you’re requesting to.

#### Input Validation and Filtering
#### 输入验证和过滤

Input validation is the first layer of defense when it comes to securing your PHP applications. User input should never be trusted thus we need to filter and validate. But first lets differentiate filtering from validation:
输入验证是在第一层防御中当要保护你的PHP应用时。用户输入绝不能相信，所以我们需要过滤和验证。当首先我们先要区分过滤和验证：

* __Filtering__ – also called sanitization. This is used for ensuring that the data is properly formatted before we try to validate. An example of filtering is removing whitespaces from a string or removing any invalid characters from an email address.
* __Validation__ – the process of making sure that the data is what you expect it to be. For example if the web form asks for the age then you expect the age to be a number so the code must validate that what is inputted in the age field is indeed a number. And not just any number. If you expect the users who will fill out the form to be between ages 20 – 40 then you must also validate that the age that was inputted falls within that range. There are lots of things to consider when validating user input, as programmers its our duty to ensure that we’ve covered most of the use cases.
* __过滤__ – 主要是防止恶意用户的输入，例如SQL注入。
* __验证__ – 主要是判断数据类型是否为我们想要的。

##### Filtering
##### 过滤

PHP comes with filtering functions that you can use to sanitize data before saving into the database.

PHP已经自带了许多过滤数据函数，以便于在存到数据库前处理敏感数据。

* __addslashes__ – adds a backslash before a single quote ('), double quote ("), and NULL byte (\).
* __filter_var__ – sanitizes strings based on the filters listed here
* __htmlspecialchars__ – converts HTML strings into their corresponding entity.
* __htmlentities__ – the same as htmlspecialchars the only difference is that * __htmlentities__ try to encode all characters which have HTML character entity equivalents. What this means is that you will have a much longer resulting string if the string that you’re trying to use contains not only HTML but also characters which has an HTML entity equivalents.
* __preg_replace__ – replaces all the string that matches the pattern that you specify.
* __strip_tags__ – strips all HTML and PHP tags from the original string.
* __trim__ – used for trimming leading and trailing whitespaces from the original string.

##### Validation
##### 验证

PHP also comes with validation functions one of those is the `filter_var`. You can use it to validate different types of data:

PHP也提供了一些验证数据类型的函数：

* __FILTER_VALIDATE_BOOLEAN__ – used for validating if the value is either true or false
* __FILTER_VALIDATE_EMAIL__ – used for validating if the value is a valid email
* __FILTER_VALIDATE_REGEXP__ – used for validating if the value matches a specific expression
* __FILTER_VALIDATE_URL__ – used for validating if the value matches the accepted pattern of a URL
* __FILTER_VALIDATE_INT__ – used for validating if the value is an integer
* __FILTER_VALIDATE_FLOAT__ – used for validating if the value is a float or a decimal number
* __FILTER_VALIDATE_IP__ – used for validating if the value is a valid IPv4 or IPv6 IP address

Here’s how to use the `filter_var` function to validate user input:
{% highlight php linenos %}
<?php
$email = filter_var($_POST['email'], FILTER_VALIDATE_EMAIL);
$age = filter_var($_POST['age'], FILTER_VALIDATE_INT);

if($email && $age && ($age >= 14 && $age <= 30)){
  //do something
}
?>
{% endhighlight %}

There are also a bunch of PHP functions that checks for a specific data type and returns true if the value meets:

这里还有其他的PHP内置函数用来判断变量的类型：

__is\_array__
__is\_bool__
__is\_double__
__is\_float__
__is\_integer__|__is\_long__|__is\_int__ – checks if value is a valid integer. Note that this doesn’t check for the data type since all user input is always in string so either the value `'1'` or simply `1` will pass.
is_null – checks if a variable is NULL
__is\_numeric__ – checks if a value is a valid number, the main difference of this function with is_int is that it also checks for the data type so string numbers such as `'1'`, `'23'`, or `'14'` will return false.
__is\_object__
__is\_resource__
__is\_scalar__
__is\_string__

And there are also those that checks for the presence of a specific value:

还有检查一个变量是否存在或为空：

* __isset__ – checks if a specific variable has been set or declared. Note that this disregards the actual value so if the variable in question doesn’t have a value assigned to it (aka undefined) then it will still return true.
* __empty__ – checks if a specific variable has a truthy value. Here’s a good reference on this subject: [type comparisons](http://www.php.net/manual/en/types.comparisons.php)

#### Working with Databases
#### 与数据库相关

##### Limit User Privileges
##### 限制用户权限

When working with databases its a good practice to not use the root user as the user of the database. Sometimes out of laziness we tend to use the default database user in MySQL when connecting to the database like this:
{% highlight php linenos %}
<?php
$db = new Mysqli("localhost", "root", "", "my_db");
?>
{% endhighlight %}
当使用数据库的时候不要在应用中使用root帐号去连接数据库，创建其他数据库用户，并限制数据库操作权限，是一个很好的选择。这样当你的应用遭受攻击时，例如SQL注入，可以最大程度保护你的数据库免受伤害。

##### Use PDO Or MySQLi
##### 使用PDO或者MySQLi

Use the PDO or MySqli extension when building applications that connect to the MySQL database. The original PHP MySQL API is already deprecated and therefore no longer recommended. Using PDO or MySqli will give you the benefit of using parametrized queries which effectively reduces the risk of SQL injection attacks if used correctly. Here’s an example on how to perform database queries using PDO:
{% highlight php linenos %}
<?php
$user_id = $_GET['id'];
if(is_int($user_id)){ //check if id is an integer

  try{
      $conn = new PDO("mysql:host=localhost;dbname=my_db", $_SERVER['db_user'], $_SERVER['db_password']);
      $conn->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION); //tell PDO to throw exceptions  
  
      $sql = $conn->prepare("SELECT username, role FROM tbl_users WHERE user_id = :user_id");
      $sql->bindParam(':user_id', $user_id, PDO::PARAM_INT); //safely substitute the placeholder(:user_id) to the real value ($_GET['id'])
      $sql->execute(); //execute the query
  
      $user = $sql->fetch();
      echo $user['username'];
  
  }catch(PDOException $e){
      log_exception($e->getMessage()); //log the exception, don't echo 
  }
}
?>
{% endhighlight %}
使用PDO或MySqli扩展来连接MySQL数据库，原本的mysql函数已经被弃用了。PDO和MySqli支持参数绑定查询，这可以减少SQL注入的风险。

How does PDO make things more secure you ask? Its more secure in the sense that it sends the query and data (user input) separately to MySQL. So what happens is that the SQL string that you supplied as the argument for the prepare method is parsed and then later on using bindParam the placeholder is safely substituted to the user input. Finally the query is executed. In simple terms MySQL considers every user input as a string with no meaning when PDO is used so SQL injection is effectively prevented.

PDO书如何保证安全的？它分开地发送查询和用户输入到MySQL。所以你所提供的SQL语句，即`prepare`中的参数和之后使用`bindParam`函数将占位符安全的替换为用户输入。最后查询被执行。MySQL认为每一个用户输入都是一段字符，没有任何意义当使用PDO的时候。所以SQL注入被阻止了。

If you want to learn more about PDO be sure to check out the [PDO tutorial for MySQL Developers](http://wiki.hashphp.org/PDO_Tutorial_for_MySQL_Developers).

#### Storing Passwords
#### 存储密码

And it might already be old news to you but these functions for hashing passwords isn’t safe either as attackers can use brute force attack or rainbow tables in order to determine a password:

下面的加密函数已经被证明能够在段时间内被破译了（PS：一位中国的女性王小云教授）

* __md5__
* __sha1__

You can use the following functions instead:

* __hash_pbkdf2__
* __crypt__
* __password_hash__

Note that some of the hashing functions like `hash_pbkdf2` and `password_hash` are only available on PHP 5.5. `crypt` is available on PHP 4 and 5.

#### Conclusion
#### 总结

We’ve barely scratch the surface with this guide. There’s a lot more you can do to improve the security of the applications that you’re writing. Be sure to check out the resources below if you want to learn more about securing PHP applications.

Resources

* [OWASP PHP Security Cheat Sheet](https://www.owasp.org/index.php/PHP_Security_Cheat_Sheet)
* [Survive the Deep End: PHP Security](http://phpsecurity.readthedocs.org/en/latest/)
* [PHP.Net Security Manual](http://www.php.net/manual/en/security.php)
* [PHP Security Guide](http://phpsec.org/)
* [PHP Security Best Practices for Sys Admins](http://www.cyberciti.biz/tips/php-security-best-practices-tutorial.html)