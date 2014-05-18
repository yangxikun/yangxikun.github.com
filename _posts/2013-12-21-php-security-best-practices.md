---
layout: post
title: "PHP安全最佳实践"
description: ""
category: PHP
tags: [PHP安全]
---
{% include JB/setup %}
翻译自：[PHP Security Best Practices](http://anchetawern.github.io/blog/2013/12/15/php-security-best-practices/)

在这一篇博文中将会探讨一些关于PHP安全的最佳实践。

#### 经常更新

如果可能，使用最新发布的稳定版PHP。因为它包含了一些安全的更新和BUG的修复。这能够让PHP应用更加安全，当然性能也会更好。

<!--more-->

#### 安全配置

* 在php.ini中，设置`expose_php = Off`：

这会使PHP的版本信息在HTTP响应头中`X-Powered-By`不会出现。一个攻击者可以利用这个信息进行漏洞攻击。（PS：在HTTP响应头中的Server字段也会暴露服务器操作系统信息和WEB服务器信息，Apache的话需要在编译的时候进行一些修改，才能改变这个字段的值）。

![img](/assets/img/201312210201.png)

* 谨慎的使用`phpinfo()`函数，不要被别人访问到！（PS：phpMyadmin也一样。）

* 记录错误，而不是暴露错误。像下面那样配置你的php.ini。

{% highlight php linenos %}
display_startup_errors = Off #disable displaying of startup errors
display_errors = Off #disable displaying of errors
html_errors = Off #disable formatting of errors in HTML
error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT #report all errors, warnings and notices including coding standards
log_errors = On #log errors to a file
{% endhighlight %}

* 当你不需要上传文件时，关闭文件上传功能。

{% highlight php linenos %}
file_uploads = Off
{% endhighlight %}

如果你的应用需要上传文件，你必须确保文件上传的安全性。这里有一篇很好的博文[关于如何使用PHP创建一个安全的文件上传](http://www.sitepoint.com/file-uploads-with-php/)。你也可以使用这个[库](https://github.com/codeguy/Upload)来处理文件上传。

* 关闭远程文件操作，建议使用curl代替它们。设置`allow_url_fopen = Off`和`allow_url_include = Off`。

{% highlight php linenos %}
allow_url_fopen = Off #disables processing of urls
allow_url_include = Off #disable including of urls to files (e.g include 'http://iamanevilfile.php')
{% endhighlight %}

* 限制POST数据的大小为你的应用所需要的。因为这可以防止攻击者对你发动洪流攻击--通过发送数据量很大的POST请求（这会导致服务器带宽被大量占用）。注意，这个值可以设置为K、M或G。

注意`post_max_size`的值要比`upload_max_filesize`大，因为POST的数据不仅包含了上传的文件，还有头信息等。

{% highlight php linenos %}
post_max_size = 10M
{% endhighlight %}

`memory_limit`也应该要比`post_max_size`大。

{% highlight php linenos %}
memory_limit = 25M
{% endhighlight %}

* 限制input时间。这会限制PHP对`$_POST` 或 `$_GET`的解析时间。

{% highlight php linenos %}
max_input_time = 5
{% endhighlight %}

* 限制最大执行时间，这个是指使用的CPU秒数。当PHP脚本运行达到这个时间时，它将会自动终止。默认值30秒在大多数情况下是足够的，所以通常情况下你不需要改变它。

{% highlight php linenos %}
max_execution_time = 30
{% endhighlight %}

* 限制对一些涉及到系统安全函数的使用，例如`exec`, `passthru`, `shell_exec`, `proc_open`, and `popen`。

* 仅仅允许PHP进程访问特定目录，该值最好是web服务器可以访问的根目录。

{% highlight php linenos %}
open_basedir = /var/www/public_html
{% endhighlight %}

* 设置`upload_tmp_dir`与`open_basedir`不同。这样可以避免上传的文件被执行。

{% highlight php linenos %}
upload_tmp_dir = /var/www/uploads/tmp
{% endhighlight %}

* 确保你的web目录只能被读。（PS：这还是得分情况的，比如你的web需要进行Log，那么得确保Log目录是可写的）

{% highlight php linenos %}
sudo chmod -R 0444 /var/www/public_html
{% endhighlight %}

#### 使用CURL

总是使用CURL向其它服务器发送请求，特别是含有敏感数据的时候。因为CURL可以发送HTTPS的请求。
{% highlight php linenos %}
<?php
$url = 'https://bitpay.com/api/invoice';
$req = curl_init($url);
curl_setopt($req, CURLOPT_RETURNTRANSFER, TRUE);
$response = curl_exec($req);
?>
{% endhighlight %}

确保以下的选项被设置，当你传输敏感数据的时候：

* `CURLOPT_SSL_VERIFYPEER` – should be set to `TRUE` always. This will tell CURL to check if the remote certificate of the server where you’re performing a request is valid.
* `CURLOPT_SSL_VERIFYHOST` – should be set to `TRUE` always. This tells CURL to check that the Certificate was issued to the entity that you’re requesting to.

#### 输入验证和过滤

当要保护你的PHP应用时，输入验证是第一层防御。用户输入绝不能相信，所以我们需要过滤和验证。首先我们先要区分过滤和验证：

* __过滤__ – 用于保证在验证之前，用户输入的数据被正确的格式化了。例如：输入的字符串中删除空格、删除email地址中的非法字符。
* __验证__ – 主要是判断数据类型是否为我们想要的。例如：如果应用请求用户输入年龄，那么就要验证用户输入的年龄字段的值是否为正整数、或者年龄字段是一个范围值（20-40）,那么你必须验证范围是否合法。作为开发者，考虑所有可能的输入是我们的职责。

##### 过滤

PHP已经自带了许多过滤数据函数，以便于在存到数据库前处理敏感数据。

* __addslashes__ – adds a backslash before a single quote \('\), double quote \("\), and NULL byte \(\).
* __filter_var__ – sanitizes strings based on the filters listed here
* __htmlspecialchars__ – converts HTML strings into their corresponding entity.
* __htmlentities__ – the same as htmlspecialchars the only difference is that * __htmlentities__ try to encode all characters which have HTML character entity equivalents. What this means is that you will have a much longer resulting string if the string that you’re trying to use contains not only HTML but also characters which has an HTML entity equivalents.
* __preg_replace__ – replaces all the string that matches the pattern that you specify.
* __strip_tags__ – strips all HTML and PHP tags from the original string.
* __trim__ – used for trimming leading and trailing whitespaces from the original string.

使用哪些函数取决于你的特定需要。如果你要保存一段字符串到数据库中，并且字符串中可能含有单引号或双引号，那么你应使用`addslashes`函数。这能确保你不会遇到非法字符错误，当插入数据的时候。

##### 验证

PHP也提供了一些验证数据类型的函数：

* __FILTER_VALIDATE_BOOLEAN__ – used for validating if the value is either true or false
* __FILTER_VALIDATE_EMAIL__ – used for validating if the value is a valid email
* __FILTER_VALIDATE_REGEXP__ – used for validating if the value matches a specific expression
* __FILTER_VALIDATE_URL__ – used for validating if the value matches the accepted pattern of a URL
* __FILTER_VALIDATE_INT__ – used for validating if the value is an integer
* __FILTER_VALIDATE_FLOAT__ – used for validating if the value is a float or a decimal number
* __FILTER_VALIDATE_IP__ – used for validating if the value is a valid IPv4 or IPv6 IP address

这里有个例子，关于使用`filter_var`函数验证用户输入：
{% highlight php linenos %}
<?php
$email = filter_var($_POST['email'], FILTER_VALIDATE_EMAIL);
$age = filter_var($_POST['age'], FILTER_VALIDATE_INT);

if($email && $age && ($age >= 14 && $age <= 30)){
  //do something
}
?>
{% endhighlight %}

这里还有其他的PHP内置函数用来判断变量的类型：

* __is\_array__  – checks if a variable contains an array.

* __is\_bool__ – checks if a variable contains a boolean value.

* __is\_double__ – checks if a variable contains a double.

* __is\_float__ – checks if a variable contains a floating point number.

* __is\_integer__ | __is\_long__ | __is\_int__ – checks if value is a valid integer. Note that this doesn’t check for the data type since all user input is always in string so either the value `'1'` or simply `1` will pass.
is_null – checks if a variable is NULL

* __is\_numeric__ – checks if a value is a valid number, the main difference of this function with is_int is that it also checks for the data type so string numbers such as `'1'`, `'23'`, or `'14'` will return false.

* __is\_object__  – checks if a variable contains an object.

* __is\_resource__ – checks if a variable contains a resource.

* __is\_scalar__ – checks if a variable contains a scalar value.

* __is\_string__ – checks if a variable contains string.

还有检查一个变量是否存在或为空：

* __isset__ – checks if a specific variable has been set or declared. Note that this disregards the actual value so if the variable in question doesn’t have a value assigned to it (aka undefined) then it will still return true.
* __empty__ – checks if a specific variable has a truthy value. Here’s a good reference on this subject: [type comparisons](http://www.php.net/manual/en/types.comparisons.php)

##### 输入过滤和验证的库

* [Respect\Validation](http://documentup.com/Respect/Validation/)
* [Filterus](https://github.com/ircmaxell/filterus)
* [Valitron](https://github.com/vlucas/valitron)
* [HTML Purifier](http://htmlpurifier.org/)

#### 与数据库相关

##### 限制用户权限

有时开发者会像下面这样去连接MySQL数据库：
{% highlight php linenos %}
<?php
$db = new Mysqli("localhost", "root", "", "my_db");
?>
{% endhighlight %}
当使用数据库的时候不要在应用中使用root帐号去连接数据库，创建其他数据库用户，并限制数据库操作权限，是一个很好的选择。这样当你的应用遭受攻击时，例如SQL注入，可以最大程度保护你的数据库免受伤害。

限制用户权限是很简单的。在下面的截屏中，我使用phpmyadmin来创建一个只有读取权限的用户：

![img](/assets/img/201312210102.png)

你也可以限制用户对资源的访问。设置合理的资源限制以降低恶意用户向你的数据库发送大量查询请求的可能性。

![img](/assets/img/201312210103.png)

限制用户的权限能有效的降低SQL注入带来的损害。这意味着如果一个攻击执行如下查询：

{% highlight php linenos %}
DROP TABLE tbl_users
{% endhighlight %}

那么该查询将无法执行成功，如果该数据库用户没有删除表的权限。但是，如果攻击者使用的是数据库管理员的帐号（例如你的web应用使用root连接数据库），那该如何防范注入攻击？这就是使用PDO和预绑定参数的原因。

##### 使用PDO或者MySQLi

使用PDO或者MySqli扩展当应用需要连接数据库的时候。原生的PHP MySQL API已经被弃用了，也不再推荐使用它。使用PDO和MySqli将会给你带来使用绑定参数查询以防止SQL注入危害的好处，如果使用得正确的话。这里有一个使用PDO的例子：

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

PDO书如何保证安全的？它分开地发送查询语句和用户输入的数据给MySQL。所以你所提供的SQL语句，即`prepare`中的参数和之后使用`bindParam`函数将占位符安全的替换为用户输入。最后查询被执行。MySQL认为每一个用户输入都是一段字符，没有任何意义当使用PDO的时候。所以SQL注入被阻止了。

如果你想要了解更多关于PDO的使用：[PDO tutorial for MySQL Developers](http://wiki.hashphp.org/PDO_Tutorial_for_MySQL_Developers).

#### 存储密码

下面的加密函数已经被证明能够在短时间内被破译了（PS：一位中国的女性王小云教授）

* __md5__
* __sha1__

你可以使用下面的函数代替它们：

* __hash_pbkdf2__
* __crypt__
* __password_hash__

注意`hash_pbkdf2`和`password_hash`在PHP 5.5才有。 `crypt`在PHP 4和5都有。

这里有几个关于如何使用上面提到的函数的例子：

{% highlight php linenos %}
<?php
$password = 'mySupeRandomPassword'; //note: don't use a password like this

//using hash_pbkdf2
$salt = mcrypt_create_iv(16, MCRYPT_DEV_URANDOM); //generate a random salt
$iterations = '1525';
$hash = hash_pbkdf2("sha256", $password, $salt, $iterations, 30); //hashing algorithm, raw password, random salt, iterations, hash length

//using crypt
$salt = mcrypt_create_iv(20, MCRYPT_DEV_URANDOM); //generate a random salt
$hash = crypt($password, $salt); //raw password, random salt


//using password_hash
$hash = password_hash($password, PASSWORD_DEFAULT); //PASSWORD_DEFAULT uses the Bcrypt alogrithm, you can also use PASSWORD_BCRYPT if you want to use the CRYPT_BLOWFISH algorithm for hashing the password
?>
{% endhighlight %}

当使用`hash_pbkdf2`时，你可以将hash结果和salt存储起来。

{% highlight php linenos %}
<?php
//verifying using hash_pbkdf2
$password = $_POST['password'];

/*
get hash and salt from database
*/

$hash = hash_pbkdf2("sha256", $password, $salt_from_db, $iterations, 30);
if($hash_from_db == $hash){
  //do something
}
?>
{% endhighlight %}

一些人认为你应该将salt和hash结果存储到不同的数据库。也许这是正确的，当你没有使用随机的salt时。攻击者很难破译密码，即使他同时拥有salt和hash结果，因为salt对于每个用户都是随机的。

对于`crypt`和`password_hash`，就不需要存储salt了，因为你可以验证密码是否合法，而不需要指定salt：

{% highlight php linenos %}
<?php
//verifying using crypt
$password = $_POST['password'];

/* 
get hash from database
*/

if(crypt($password, $hash) == $hash){ //check if password is valid
  //do something
}


//verifying using password hash
if(password_verify($password, $hash)){
  //do something
}
?>
{% endhighlight %}

注意，你也可以使用`password_verify`函数验证使用`crypt`和`password_hash`加密的密码。因为它们都使用[C Crypt Scheme](http://en.wikipedia.org/wiki/Crypt_(C)。

你也可以使用密码加密库：[PHPAss](https://github.com/hautelook/phpass/)和[Password-Compat](https://github.com/ircmaxell/password_compat)。使用这些库的好处是因为它们通常兼容地版本的PHP，并且是安全的。这里有两个例子：

{% highlight php linenos %}
<?php
//using password-compat
require 'vendor/ircmaxell/password-compat/lib/password.php';
$hash = password_hash($password, PASSWORD_BCRYPT);

//verifying
if(password_verify($password, $hash)){
  //do something
}
?>
{% endhighlight %}

{% highlight php linenos %}
<?php
//using PHPAss
$cost = 8; //algorithmic cost that should be used, you can play around this value but this is mostly dependent on your servers hardware
$portable_hash = false; //do not store salts along with hash
$phpass = new PasswordHash($cost, $portable_hash);

$hash = $phpass->HashPassword($password);

//verifying
if($phpass->CheckPassword($password, $hash)){
  //do something
}
?>
{% endhighlight %}

请注意password-compact库使用像PHP 5.5中的`password_hash`同样的语法。但这个库只能运行在PHP 5.3.7及以上。所以这个库旨在保证PHP版本低于5.5的向前兼容性。这意味着如果你使用的是PHP 5.5，那么就没必要使用它了。

其它需要注意的事项：

* 当用户忘记密码时，不要发送旧的密码给用户，仅通过email发送一个修改密码的链接。
* 不要记录用户未加密的密码，只存储加密后的密码。
* 使用随机的salt
* 鼓励用户使用复杂的密码，并在用户注册的时候，显示密码安全强度。

#### 处理文件上传

不要使用超全局变量`$_FILE`来检查文件类型，因为文件扩展名很容易造假：

{% highlight php linenos %}
<?php
if($_FILES["file"]["type"] == 'jpg'){
  //do something with the file
}
?>
{% endhighlight %}

使用`finfo`类检查实际的文件MIME类型。虽然这会使检查慢点，但是值得的：

{% highlight php linenos %}
<?php
$file_info = new finfo(FILEINFO_MIME_TYPE);
$file_contents = file_get_contents($_FILES['iamnotanevilfile']['tmp_name']);
$mime_type = $file_info->buffer($file_contents);
//this will return any valid mime type listed here: http://en.wikipedia.org/wiki/Internet_media_type
?>
{% endhighlight %}

最好使用一个专门处理这类工作的库，例如[upload library](https://github.com/codeguy/Upload)。这里有一个例子，验证是否为图片文件，且大小不超过2MB：

{% highlight html linenos %}
<form method="POST" enctype="multipart/form-data">
    <input type="file" name="some_file" value=""/>
    <input type="submit" value="Upload File"/>
</form>
{% endhighlight %}

{% highlight php linenos %}
<?php
$upload_path = new \Upload\Storage\FileSystem('/upload_path');
$file = new \Upload\File('some_file', $upload_path);

$image_types = array('image/gif', 'image/png', 'image/jpeg', 'image/bmp');

$file->addValidations(array(
    new \Upload\Validation\Mimetype($image_types), //can also supply a string
    new \Upload\Validation\Size('2M') //size should be 2 MB or less, you can also use B, K, G as the size unit
));

//try to upload the file
try{
    $file->upload(); //the file is uploaded if it successfully pass through the validation
}catch(\Exception $e){
    $errors = $file->getErrors(); //the file upload failed
}
?>
{% endhighlight %}

#### 总结

在这篇文章中，你已经学习了一些基本的方法来保护你的PHP项目。但还有许多方法来提高你的应用的安全度，如果你希望学习到更多关于PHP安全的知识，请参考下面的资源。

#### PHP安全相关资源

* [OWASP PHP Security Cheat Sheet](https://www.owasp.org/index.php/PHP_Security_Cheat_Sheet)
* [Survive the Deep End: PHP Security](http://phpsecurity.readthedocs.org/en/latest/)
* [PHP.Net Security Manual](http://www.php.net/manual/en/security.php)
* [PHP Security Guide](http://phpsec.org/)
* [PHP Security Best Practices for Sys Admins](http://www.cyberciti.biz/tips/php-security-best-practices-tutorial.html)