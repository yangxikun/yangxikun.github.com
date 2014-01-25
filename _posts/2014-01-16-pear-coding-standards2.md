---
layout: post
title: "Pear编码标准 10、13"
description: ""
category: PHP
tags: [PHP编码规范]
---
{% include JB/setup %}
*翻译自[Coding Standards](http://pear.php.net/manual/en/standards.php)的10、13小节*

#### 文件头注释
- - -
在PEAR库的所有源代码文件都要包含一个“页面级”的注释在文件开头和每一个类的都要有“类级别”的注释。下面有一个例子：
{% highlight php linenos %}
<?php

/* vim: set expandtab tabstop=4 shiftwidth=4 softtabstop=4: */

/**
 * 关于本文件短的描述
 *
 * 详细的描述（如果需要的话……）
 *
 * PHP version 5版本
 *
 * 许可声明
 * LICENSE: This source file is subject to version 3.01 of the PHP license
 * that is available through the world-wide-web at the following URI:
 * http://www.php.net/license/3_01.txt.  If you did not receive a copy of
 * the PHP License and are unable to obtain it through the web, please
 * send a note to license@php.net so we can mail you a copy immediately.
 *
 * @category   CategoryName
 * @package    PackageName
 * @author     Original Author <author@example.com>
 * @author     Another Author <another@example.com>
 * @copyright  1997-2005 The PHP Group
 * @license    http://www.php.net/license/3_01.txt  PHP License 3.01
 * @version    SVN: $Id$
 * @link       http://pear.php.net/package/PackageName
 * @see        NetOther, Net_Sample::Net_Sample()
 * @since      File available since Release 1.2.0
 * @deprecated File deprecated in Release 2.0.0
 */

/*
* 将includes，constant defines和$_GLOBAL设置写在这里。
* 确保他们有合适的文档注释，以免phpDocumentor将它们当作“页面级”注释处理。
*/

/**
 * 关于类的简短描述
 *
 * 详细的描述（如果需要的话……）
 *
 * @category   CategoryName
 * @package    PackageName
 * @author     Original Author <author@example.com>
 * @author     Another Author <another@example.com>
 * @copyright  1997-2005 The PHP Group
 * @license    http://www.php.net/license/3_01.txt  PHP License 3.01
 * @version    Release: @package_version@
 * @link       http://pear.php.net/package/PackageName
 * @see        NetOther, Net_Sample::Net_Sample()
 * @since      Class available since Release 1.2.0
 * @deprecated Class deprecated in Release 2.0.0
 */
class Foo_Bar
{
}

?>
{% endhighlight %}
<!--more-->

#### 具有可变内容的标签
- - -
简短的描述

简短的描述必须包含在所有文档块中。他们应该是简洁的句子，而不是名称。请阅读Coding Standard's [Sample File](http://pear.php.net/manual/en/standards.sample.php) 关于怎样写好描述。

PHP 版本

以下的其中一个必须包含在页面级文档块：
{% highlight php linenos %}
* PHP version 4
* PHP version 5
* PHP versions 4 and 5
{% endhighlight %}

@license

这里有几种可能的许可声明。以下之一必须被放置在页面级和类级别的文档块：
{% highlight php linenos %}
* @license   http://www.apache.org/licenses/LICENSE-2.0  Apache License 2.0
* @license   http://www.freebsd.org/copyright/freebsd-license.html  BSD License (2 Clause)
* @license   http://www.debian.org/misc/bsd.license  BSD License (3 Clause)
* @license   http://www.freebsd.org/copyright/license.html  BSD License (4 Clause)
* @license   http://www.opensource.org/licenses/mit-license.html  MIT License
* @license   http://www.gnu.org/copyleft/lesser.html  LGPL License 2.1
* @license   http://www.php.net/license/3_01.txt  PHP License 3.01
{% endhighlight %}
欲了解更多信息，请参见PEARGroup's [Licensing Announcement](http://pear.php.net/group/docs/20040402-la.php)。

@link

下面的链接必须被包含在页面级和类级别的文档块。当然，修改“PackageName”为你的包名。这保证在生成的文档链接能跳转到你的包。
{% highlight php linenos %}
* @link      http://pear.php.net/package/PackageName
{% endhighlight %}

@author

这里没有硬性规定当一个新的代码贡献者应该加入作者名单中对于给定的源文件。通常来说，他们的改变应该分为“实质性”的类别（意味着某些地方有大约10%到20%的代码变更）。Exceptions应该通过重写函数或者提供新的逻辑实现。

简单的代码重组或错误修复将不足以增加了一个新的名字到作者名单中。

@since

当一个文件或类在一个初始化的代码版本后增加，就需要这个标签。不要在初始版本中使用。

@deprecated

当一个文件或类不再使用时，为了向后兼容，需要这个标签。

#### 可选的标签
- - -
@copyright

根据自己的需要进行版权声明。当格式化这个标签时，年份应该要由四个数字组成。如果涉及几年的时间跨度，在最早和最晚的年份之间使用著作权人可以是你、名单、公司、PHP组织等。例如：连字符。
{% highlight php linenos %}
* @copyright 2003 John Doe and Jennifer Buck
* @copyright 2001-2004 John Doe
* @copyright 1997-2004 The PHP Group
* @copyright 2001-2004 XYZ Corporation
{% endhighlight %}
许可摘要

如果您使用的是PHP的许可证，请使用上面提供的摘要文本。如果正在使用其它许可证，请删除PHP的许可摘要。为了简单，可以直接使用`LICENSE: `作为许可摘要。

@see

添加@see标签，当你想引导用户到文档的其他章节。当你有多个参考章节时，用逗号隔开它们，而不是增加多个@see标签。

#### 顺序和间距
- - -
为了使源代码具有长期的可读性，文本和标签必须符合上面例子中提供的顺序和间距。这个标准也被JavaDoc标准采纳。

#### @package_version@ 用法
- - -
这里用两种方法实现@package_version@替换。该过程取决于你自己写的`package.xml`文件，或者使用的`PackageFileManager`。

对于那些直接授权的`package.xml`文件，增加一个`<replace>`。XML格式例如：
{% highlight php linenos %}
<file name="Class.php">
  <replace from="@package_version@" to="version" type="package-info" />
</file>
{% endhighlight %}
使用`PackageFileManager`的维护者需要为每个文件调用`addReplacement()`：
{% highlight php linenos %}
<?php
$pkg->addReplacement('filename.php', 'package-info',
                     '@package_version@', 'version');
?>
{% endhighlight %}

#### 命名约定

#### 全局变量和函数
- - -
如果你的包需要定义全局变量，他们的名字应该以单下划线开头，后跟软件包名称和另一个下划线。例如，PEAR包采用了一个名为`$_PEAR_destructor_object_list`全局变量。
全局函数应该以“驼峰型”风格命名。另外，还要以包名开头，以避免不同包之间命名的冲突。包名后跟的单词首字母要小写，之后每一个单词的首字母要大写。例如：<br>
`XML_RPC_serializeData()`<br>

#### 类
- - -
类应当给予描述性的名称。避免使用缩写。类名应该总是以一个大写字母开头。PEAR的类层次结构还体现在类名，每个层次有一个下划线隔开。良好的类名的例子有：<br>
`Log` `Net_Finger` `HTML_Upload_Error`<br>

#### 类的变量和方法
- - -
类的变量（也称为属性）和方法应该使用“驼峰型”风格命名。例如（这些是公有成员）：<br>
`$counter` `connect()` `getData()` `buildSomeWidget()`<br>
私有的类成员名称以一个下划线开头，例如：<br>
`$_status` `_sort()` `_initTree()`<br>
受保护的类成员前面没有一个下划线。例如：<br>
`protected $somevar` `protected function initTree()`<br>

#### 常量
- - -
常量总是应该全部大写，用下划线分隔单词。使用包名或者类名作为常量名称的前缀。例如：<br>
`DB_DATASOURCENAME` `SERVICES_AMAZON_S3_LICENSEKEY`<br>
注意对于`true`、`false`、`null`这三个常量是例外，它们应该保持小写。
