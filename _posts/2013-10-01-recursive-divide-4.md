---
layout: post
title: "递归与分治--合并排序"
description: ""
category: 算法
tags: [算法]
---
{% include JB/setup %}

###思想

>将待排序元素分成大小大致相同的两个子集合,分别对两个子集合进行排序,最终将排好序的子集合合并成所要求的排好序的集合.

>递归的归并排序

{% highlight cpp linenos %}
void MergeSort(Type a[], int left, int right){
    if (left < right ) {//至少要有2个元素
        int i = (left+right)/2;
        MergeSort(a, left, i);//左半部分进行归并排序
        MergeSort(a, i+1, right);//右半部分进行归并排序
        Merge(a, b, left, i, right);//将a[left...i]和a[i+1...right]合并到辅助数组b中
        Copy(a, b, left, right);//因为归并排序不是原址排序,所以要复制回数组a
    }
}
{% endhighlight %}

>归并排序非递归实现

>*需要设置一个归并的步长,起始为1,到n/2结束*

{% highlight cpp linenos %}
void MergeSort(Type a[], int n){
    Type *b = new Type[n];
    int s = 1;//增长步长,每次都以自身两倍增长
    while(s < n){
        MergePass(a, b, s, n);//归并到数组b中
        s += s;
        MergePass(b, a, s, n);//归并到数组a中
        s += s;
    }
}

void MergePass(Type x[], Type y[], int s, int n){
    //对数组x合并n/s个大小为s的相邻子数组到数组y中
    int i = 0;
    while (i < n) {
        Merge(x, y, i, i+s-1, i+2*s-1);//合并大小为s的相邻两段子数组
        i = i+2*s;
    }
    //剩下的元素个数少于2s
    if (i+s < n) Merge(x , y, i, i+s-1, n-1);
    else for (int j = i; j < n; j++) y[j] = x[j];
}

void Merge(Type c[], Type d[], int l, int m, int r){
    //合并c[l:m]和c[m+1:r]到d[l:r]
    int i = l, j = m+1, k = l;
    while (i <= m && j <= r) {
        if (c[i] <= c[j]) d[k++] = c[i++];
        else d[k++] = c[j++];
    }
    if (i > m) while (j <= r) d[k++] = c[j++];
    else while(i <= m) d[k++] = c[i++];
}
{% endhighlight %}