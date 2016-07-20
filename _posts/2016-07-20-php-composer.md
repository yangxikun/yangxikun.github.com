---
layout: post
title: "PHP Composer类自动加载工作原理"
description: "PHP Composer类自动加载工作原理"
category: PHP
tags: [composer]
---
{% include JB/setup %}
#### composer.json
- - -

首先需要了解下Composer的配置文件，在composer.json中，与类加载相关的指令有：

* require
* require-dev(root-only)
* autoload
* autoload-dev(root-only)
* include-path
* target-dir

每个指令在[The composer.json Schema](https://getcomposer.org/doc/04-schema.md)中都有说明。

`root-only`指在我们正在开发的项目中定义了composer.json（里面声明了项目依赖的包等），那么这个项目就是一个根包。`require-dev`和`autoload-dev`只在根包起作用。

`require`定义了项目依赖的包，这些包会被安装到与composer.json同级的vendor目录下。例如我们的项目A->B、C，而B->D，那么A是根包，B、C、D都会被安装到vendor下。

<!--more-->

`autoload`为自动加载器定义了类名到类文件的映射关系。

1、PSR-4：命名空间到类文件路径的映射。

假如根包的composer.json配置如下：

```json
{
    ……
    "require": {
        "foo/bar": "2.*"
    },
    "autoload": {
        "psr-4": {
            "App\\": "app/"
        }
    },
    ……
}
```
`foo/bar`包的composer.json配置如下：

```json
{
    ……
    "autoload": {
        "psr-4": {
            "TTD\\": "src/TTD"
        }
    },
    ……
}
```
那么类目录结构会是：

```json
/* 注释为根包项目中使用的类名 */
|-composer.json
|-app
| |-Takk.php            /* App\Takk */
| |-BAA
| | |-Uk.php            /* App\BAA\Uk */
|
|-vendor
| |-foo
| | |-bar
| | | |-composer.json
| | | |-src
| | | | |-TTD
| | | | | |-Kok.php     /* TTD\Kok */
```

2、PSR-0：命名空间到类文件路径的映射、带\_的类名到类文件路径的映射，相比psr-4，多了以命名空间作为层级目录（目录层级更深），以及对`_`的支持。

假如根包的composer.json配置如下：

```json
{
    ……
    "require": {
        "foo/bar": "2.*"
    },
    "autoload": {
        "psr-0": {
            "Aaa\\Bbb\\": "src/",
            "Ccc_Ddd_": "tsrc/"
        }
    },
    ……
}
```
`foo/bar`包的composer.json配置如下：

```json
{
    ……
    "autoload": {
        "psr-0": {
            "TTD\\": "src/TTD",
            "Eee_Fff_": "src/EF"
        }
    },
    ……
}
```
那么类目录结构会是：

```json
/* 注释为根包项目中使用的类名 */
|-composer.json
|-src
| |-Aaa
| | |-Bbb
| | | |-Jkd.php             /* Aaa\Bbb\Jkd */
| |-Ccc
| | |-Ddd
| | | |-Jkd.php             /* Ccc_Ddd_Jkd */
|-vendor
| |-foo
| | |-bar
| | | |-src
| | | | |-TTD
| | | | | |-TTD
| | | | | | |-Ipl.php       /* TTD\Ipl */
| | | |-src
| | | | |-EF
| | | | | |-Eee
| | | | | | |-Fff
| | | | | | | |-Jud.php     /* Eee_Fff_Jud */
```

3、classmap：扫描目录下所有`.php`或`.inc`文件中的类，建立类名和类文件的映射，以路径层级作为命名空间。

假如根包的composer.json配置如下：

```json
{
    ……
    "require": {
        "foo/bar": "2.*"
    },
    "autoload": {
        "classmap": ["App"]
    },
    ……
}
```
`foo/bar`包的composer.json配置如下：

```json
{
    ……
    "autoload": {
        "classmap": ["Uuu/Xxx"]
    },
    ……
}
```
那么类目录结构会是：

```json
/* 注释为根包项目中使用的类名 */
|-composer.json
|-App
| |-Htt
| | |-Koo.php           /* Htt\Koo */
|-vendor
| |-foo
| | |-bar
| | | |-Uuu
| | | | |-Xxx
| | | | |-Ppp
| | | | | | |-Kuu.php       /* Ppp\Kuu */
```

4、files：用于加载一些php函数。

假如根包的composer.json配置如下：

```json
{
    ……
    "require": {
        "foo/bar": "2.*"
    },
    "autoload": {
        "files": ["src/Aaa/Hhh/K.php"]
    },
    ……
}
```
`foo/bar`包的composer.json配置如下：

```json
{
    ……
    "autoload": {
        "files": ["src/Yyy/Fd.php"]
    },
    ……
}
```
那么类目录结构会是：

```json
/* 注释为根包项目中使用的类名 */
|-composer.json
|-src
| |-Aaa
| | |-Hhh
| | | |-K.php
|-vendor
| |-foo
| | |-bar
| | | |-src
| | | | |-Yyy
| | | | | |-Fd.php
```

#### autoload.php
- - -

在根包composer.json所在目录执行`composer install`或`composer dumpautoload`后会在vendor目录下产生autoload.php和composer目录，那么我们的项目中只需要require这个文件autoload.php，就可以使用到上文分析出来的各种加载方式产生的类。

vendor目录结构：

```json
|-vendor
|-autoload.php
| |-composer
| | |-autoload_psr4.php
| | |-autoload_namespaces.php
| | |-autoload_classmap.php
| | |-autoload_files.php
| | |-autoload_real.php
| | |-autoload_static.php
| | |-ClassLoader.php
| | |-installed.json
| | |-LICENSE
/* 其他包 */
```

直接跟踪autoload.php中的代码执行，可以了解到其流程主要就是实例化一个单例的\Composer\Autoload\ClassLoader，然后初始化其私有成员变量（根据PHP版本和是否为HHVM其初始化方式不同），之后调用spl\_autoload\_register注册\Composer\Autoload\ClassLoader::loadClass()方法到SPL \_\_autoload函数队列中，最后逐个包含composer.json.autoload.files配置的文件。

其实看到autoload_real.php这个文件中ComposerAutoloaderInit\*\*\*::getLoader()方法，对其代码有些疑惑的：

1、为啥 \Composer\Autoload\ClassLoader的实例化不直接`require __DIR__ . '/ClassLoader.php';`？

```php
<?php
public static function loadClassLoader($class)
{
    public static function loadClassLoader($class)
    {
        if ('Composer\Autoload\ClassLoader' === $class) {
            require __DIR__ . '/ClassLoader.php';
        }
    }

    public static function getLoader()
    {
        if (null !== self::$loader) {
            return self::$loader;
        }

        spl_autoload_register(array('ComposerAutoloaderInitcdf2e5379301041a36605f84abb3abc4', 'loadClassLoader'), true, true);
        self::$loader = $loader = new \Composer\Autoload\ClassLoader();
        spl_autoload_unregister(array('ComposerAutoloaderInitcdf2e5379301041a36605f84abb3abc4', 'loadClassLoader'));
        ……
    }
}
```

2、根据PHP版本和是否为HHVM，对\Composer\Autoload\ClassLoader私有成员变量的初始化方式不同，其实看到autoload_static.php中的内容就想知道为何不在composer产生这些文件的时候，直接把结果赋值给\Composer\Autoload\ClassLoader私有成员变量？

```php
<?php
    ……
    public static function getLoader()
    {
        $useStaticLoader = PHP_VERSION_ID >= 50600 && !defined('HHVM_VERSION');
        if ($useStaticLoader) {
            require_once __DIR__ . '/autoload_static.php';

            call_user_func(\Composer\Autoload\ComposerStaticInitcdf2e5379301041a36605f84abb3abc4::getInitializer($loader));
        } else {
            $map = require __DIR__ . '/autoload_namespaces.php';
            foreach ($map as $namespace => $path) {
                $loader->set($namespace, $path);
            }

            $map = require __DIR__ . '/autoload_psr4.php';
            foreach ($map as $namespace => $path) {
                $loader->setPsr4($namespace, $path);
            }

            $classMap = require __DIR__ . '/autoload_classmap.php';
            if ($classMap) {
                $loader->addClassMap($classMap);
            }
        }
        ……
    }
    ……
```

#### 类的查找
- - -

根据已注册的SPL \_\_autoload函数，调用了\Composer\Autoload\ClassLoader的`loadClass()->findFile()`。在findFile中，查找顺序是：`classmap->psr4->psr0`，找到的话就返回类文件的路径名。