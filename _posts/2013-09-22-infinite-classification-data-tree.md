---
layout: post
title: "无限分类数据的树形格式化"
description: ""
category: PHP
tags: [PHP练手]
---
{% include JB/setup %}

*转载自:[9行代码太浪费了，5行代码足矣，不用递归实现无限分类数据的树形格式化](http://www.oschina.net/code/snippet_173183_11767)*

<!--more-->
{% highlight php linenos %}
<meta charset="utf-8">
<?php

/**
 * 此方法由@Tonton 提供
 * http://my.oschina.net/u/918697
 * @date 2012-12-12 
 */
function genTree5($items) { 
    foreach ($items as $item) 
        $items[$item['pid']]['son'][$item['id']] = &$items[$item['id']]; 
    return isset($items[0]['son']) ? $items[0]['son'] : array(); 
} 

/**
 * 将数据格式化成树形结构
 * @author Xuefen.Tong
 * @param array $items
 * @return array 
 */
function genTree9($items) {
    $tree = array(); //格式化好的树
    foreach ($items as $item)
        if (isset($items[$item['pid']]))
            $items[$item['pid']]['son'][] = &$items[$item['id']];
        else
            $tree[] = &$items[$item['id']];
    return $tree;
}

$items = array(
    1 => array('id' => 1, 'pid' => 0, 'name' => '江西省'),
    2 => array('id' => 2, 'pid' => 0, 'name' => '黑龙江省'),
    3 => array('id' => 3, 'pid' => 1, 'name' => '南昌市'),
    4 => array('id' => 4, 'pid' => 2, 'name' => '哈尔滨市'),
    5 => array('id' => 5, 'pid' => 2, 'name' => '鸡西市'),
    6 => array('id' => 6, 'pid' => 4, 'name' => '香坊区'),
    7 => array('id' => 7, 'pid' => 4, 'name' => '南岗区'),
    8 => array('id' => 8, 'pid' => 6, 'name' => '和兴路'),
    9 => array('id' => 9, 'pid' => 7, 'name' => '西大直街'),
    10 => array('id' => 10, 'pid' => 8, 'name' => '东北林业大学'),
    11 => array('id' => 11, 'pid' => 9, 'name' => '哈尔滨工业大学'),
    12 => array('id' => 12, 'pid' => 8, 'name' => '哈尔滨师范大学'),
    13 => array('id' => 13, 'pid' => 1, 'name' => '赣州市'),
    14 => array('id' => 14, 'pid' => 13, 'name' => '赣县'),
    15 => array('id' => 15, 'pid' => 13, 'name' => '于都县'),
    16 => array('id' => 16, 'pid' => 14, 'name' => '茅店镇'),
    17 => array('id' => 17, 'pid' => 14, 'name' => '大田乡'),
    18 => array('id' => 18, 'pid' => 16, 'name' => '义源村'),
    19 => array('id' => 19, 'pid' => 16, 'name' => '上坝村'),
);
echo '<pre>';
print_r(genTree5($items));
print_r(genTree9($items));
echo '</pre>'
?>
{% endhighlight %}