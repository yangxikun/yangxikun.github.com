---
layout: post
title: "快速排序"
description: ""
category: 算法
tags: [算法]
---
{% include JB/setup %}

#### 思想
主要是递归思想的3个主要步骤
分解:将待排序数组a,以a\[q\]为基准元素将a\[p:r\]划分成3段a\[p:q-1\],a\[q\],a\[q+1,r\],使得a\[p:q-1\]中的所有元素小于a\[q\],而a\[q+1:r\]中任何一个元素大于等于a\[q\].

递归求解:通过递归调用快速排序算法分别对a\[p:q-1\]和a\[q+1:r\]进行排序.

合并:由于是原址排序,不需要进行合并.

<!--more-->

#### 分解

基准元素是随机的,这样是为了降低划分极不平衡的情况发生的概率

{% highlight cpp linenos %}
int RandomizedPartition (Type a[], int p, int r){
    int i = Random(p, r);
    swap(a[i], a[p]);
    pivotkey = a[p];
    while (p < r){
        while(p < r && a[r] >= pivotkey) --r;
        swap(a[p], a[r]);
        while(p < r && a[p] < pivotkey ) ++p;
        swap(a[p], a[r]);
    }
    return p;
}//RandomizedPartition
{% endhighlight %}

#### 递归求解
{% highlight cpp linenos %}
void RandomizedQuickSort(Type a[], int p, int r){
    if (p < r){//注意递归必须要有递归出口
        int q = RandomizedPartition(a, p, r);
        RandomizedQuickSort(a, p, q-1);
        RandomizedQuickSort(a, q+1, r);
    }
}
{% endhighlight %}