---
layout: post
title: "递归与分治--全排列"
description: ""
category: 算法
tags: [算法]
---
{% include JB/setup %}

###排列问题

>设R={r1,r2,r3,...,rn}是要进行排列的n个元素,Ri=R-{ri}.集合X的全排列记为Perm\(X\).\(ri\)Perm\(X\)表示在全排列Perm\(X\)的每一个排列前加上前缀ri得到的排列.R的全排列可归纳定义如下:

>当n=1时,Perm\(R\)=\(r\),其中r是集合R中唯一的元素;
>当n>1时,Perm\(R\)由\(r1\)Perm\(R1\),\(r2\)Perm\(R2\),...,\(rn\)Perm\(Rn\)构成,如此,Perm\(R1\),...,Perm\(Rn\)也可以递归分解下去.

<!--more-->
###算法代码
{% highlight cpp linenos %}
void Perm(Type list[], int k, int m){
    //产生list[k:m]的所有全排列
    if (k == m) {
        //剩下最后一个元素
        for(i = 0; i <= m; i++) cout<<list[i];
        cout<<endl;
    } else {
        for (i = k; i <= m; i++) {
            swap(list[k], list[i]);
            Perm(list, k+1, m);
            swap(list[k], list[i]);
        }
    }
}
{% endhighlight %}