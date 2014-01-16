---
layout: post
title: "动态规划--最长公共子序列"
description: ""
category: 算法
tags: [算法]
---
{% include JB/setup %}

>子序列:在此定义为一段给定序列中按顺序出现的字符序列,为原序列的子序列.例如:序列{A,B,C}的子序列可以是{A,C}

>最长公共子序列:给定两个序列,找出它们的公共子序列.

<!--more-->
###分析最优子结构

>设序列X={x1,x2,...,xm}和Y={y1,y2,...,ym}的最长公共子序列为Z={z1,z2,...,zk},则

>1.若xm=yn,则zk=xm=yn,且Zk-1是Xm-1和Yn-1的最长公共子序列
>2.若xm!=yn且zk!=xm,则Z是Xm-1和Y的最长公共子序列
>3.若xm!=yn且zk!=yn,则Z是X和Yn-1的最长公共子序列

>其中,Xm-1={x1,x2,...,xm-1};Yn-1={y1,y2,...,yn-1};Zk-1={z1,z2,...,zk-1}.

>用反证法证明上文1的结论:若zk!=xm,则{z1,z2,...,zk,xm}是X和Y的长度为k+1的公共子序列.这与Z是X和Y的最长公共子序列矛盾.所以zk=xm=yn,由此可知Zk-1是Xm-1和Yn-1的长度为k-1的最长公共子序列.上文2和3的结论也可以用反正法证明.

###建立递归关系

>设c\[i\]\[j\]记录序列Xi和Yj的最长公共子序列的长度.其中,Xi={x1,x2,...,xi};Yj={y1,y2,...,yj}.由最优子结构可得:

>>c\[i\]\[j\] = 0   i=0,j=0

>>c\[i\]\[j\] = c\[i-1\]\[j-1\]+1    i,j>0,xi=yj

>>c\[i\]\[j\] = max{c\[i\]\[j-1\],c\[i-1\]\[j\]}    i,j>0,xi!=yj

###算法代码

{% highlight cpp linenos %}
void LCSLength(int m, int n, cha *x, char *y, int **c, int **b){
    int i,j;
    for(i = 1; i <= m; i++) c[i][0] = 0;
    for(i = 1; i <= n; i++) c[0][i] = 0;
    for(i = 1; i <= m; i++)
        for(j = 1; j <= n; j++){
            //b[i][j]用于记录zk的位置,为1时表示当前就是,为2表示在Xi-1和Yj中,为3表示在Xi和Yj-1中
            if (x[i] == y[j]) {c[i][j] = c[i-1][j-1]+1; b[i][j] = 1;}
            else if (c[i-1][j] >= c[i][j-1]) {c[i][j] = c[i-1][j]; b[i][j] = 2;}
            else {c[i][j] = c[i][j-1]; b[i][j] = 3;}
    }
}

void LCS(int i, int j, char *x, int **b){
    if(i == 0 || j == 0) return;
    if(b[i][j] == 1) {LCS(i-1, j-1, x, b); cout<<x[i];}
    else if(b[i][j] == 2) {LCS(i-1, j, x, b);}
    else LCS(i, j-1, x, b);
}
{% endhighlight %}