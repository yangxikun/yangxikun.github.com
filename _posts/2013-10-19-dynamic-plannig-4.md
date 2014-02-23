---
layout: post
title: "动态规划--0-1背包问题"
description: ""
category: 算法
tags: [算法]
---
{% include JB/setup %}

#### 0-1背包问题

给定n中物品和一背包。物品i的重量是wi，其价值为vi，背包的容量为c。应如何选择装入背包中的物品，使得装入背包中物品的总价值最大？

<!--more-->
#### 定义数据结构

给定![dynamic1](/assets/img/201310220101.png)，要求找出一个n元0-1向量![dynamic1](/assets/img/201310220102.png)，使得![dynamic1](/assets/img/201310220103.png)，而且![dynamic1](/assets/img/201310220104.png)达到最大值。即求:

![dynamic1](/assets/img/201310220105.png)

![dynamic1](/assets/img/201310220106.png)

#### 最优子结构

设`(y1,y2,...,yn)`是所给0-1背包问题的一个最优解，则`(y2,y3,...,yn)`是下面所给相应子问题的一个最优解:

![dynamic1](/assets/img/201310220107.png)

![dynamic1](/assets/img/201310220108.png)

若不然,设`(z2,z3,...,zn)`是上述子问题的一个最优解,而`(y2,y3,...,yn)`不是它的最优解.由此可知,![dynamic1](/assets/img/201310220109.png)且![dynamic1](/assets/img/201310220110.png).因此![dynamic1](/assets/img/201310220111.png)

这说明`(y1,z2,...zn)`是所给0-1背包问题的一个更优解,从而`(y1,y2,...,yn)`不是所给0-1背包问题的最优解,与假设矛盾.

#### 递归关系

设所给0-1背包问题的子问题

![dynamic1](/assets/img/201310220112.png)

![dynamic1](/assets/img/201310220113.png)

的最优值为`m(i,j)`,即`m(i,j)`是背包容量为j,可选择物品为i,i+1,...,n时0-1背包问题的最优值.由0-1背包问题的最优子结构性质,可以建立计算`m(i,j)`的递归式如下:

![dynamic1](/assets/img/201310220114.png)

#### 算法代码

{% highlight cpp linenos %}
void Knapsack(Type v, int w, int c, int n, Type **m)
{
    int jMax = min(w[n]-1, c);
    for (int j = 0; j <= jMax; j++) m[n][j] = 0;
    for (int j = w[n]; j <= c; j++) m[n][j] = v[n];
    for (int i = n-1; i > 1; i--) {
        jMax = min(w[i]-1, c);
        for (int j = 0;j <= jMax; j++) m[i][j] = m[i+1][j];
        for (int j = w[i]; j <= c; j++) m[i][j] = max(m[i+1][j], m[i+1][j-w[i]]+v[i]);
    }
    if (c >= w[1]) m[1][c] = max(m[2][c], m[2][c-w[1]]+v[1]);
}

void TrackBack(Type **m, int w, int c, int n, int x)
{
    for (int i = 1; i < n; i++)
        if (m[i][c] == m[i+1][c])
            x[i] = 0;
        else {
            x[i] = 1; 
            c -= w[i];
        }
    x[n] = (m[n][c]) ? 1 : 0;
}
{% endhighlight %}