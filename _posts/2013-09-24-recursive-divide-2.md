---
layout: post
title: "递归与分治--汉诺塔"
description: ""
category: 算法
tags: [算法]
---
{% include JB/setup %}

#### 汉诺塔移动圆盘的递归分解

如下图所示:

![hanoi](/assets/img/201309230201.png)

当n=1时,直接将圆盘移到b即可;

当n>1时,首先将n-1个较小的圆盘借助b移到c上，再将a剩下的最大圆盘移到b上，接着将c中n-1个圆盘借助a移动到b。

<!--more-->

#### 算法代码
{% highlight cpp linenos %}
void hanoi(int n, int a, int b, int c)
{
    //函数实现将a中的圆盘借助c,移动到b
    if (n > 0) {
        hanoi(n-1, a, c, b);//将a中的n-1个圆盘借助b,移动到c
        move(a, b);//将a中的最后一个圆盘移动到b
        hanoi(n-1, c, b, a);//将c中的n-1个圆盘借助a,移动到b
    }
}//hanoi
{% endhighlight %}