---
layout: post
title: "哈希表"
description: ""
category: 算法
tags: [数据结构]
---
{% include JB/setup %}

#### 定义

根据设定的哈希函数H\(key\)和处理冲突的方法将一组关键字映像到一个有限的连续的地址集\(区间\)上,并以关键字在地址集中的值作为记录在表中的存储位置.

哈系函数指将key经过计算得出的结果作为哈希表的索引.

普通的哈希构造函数:直接定址法,除留取余法等

处理冲突的方法指当不同的key经过哈希函数计算后得到相同的值时,该如何处理.

常见的处理哈希冲突的方法:开放定址法\(线性探测再散列,二次探测再散列,随机探测再散列\),再哈希法,链地址法等

哈希表装填因子

<!--more-->
#### 哈希表的查找和插入
{% highlight cpp linenos %}
//开放定址哈希表的存储结构
int hashsize[] = { 997, ... };//哈希表容量递增表,一个合适的素数序列
typedef struct{
    ElemType *elem;//数据元素存储基址,动态分配数组
    int count;//当前数据元素个数
    int sizeIndex;//hashsize[sizeindex]为当前容量
}HashTable;

#define SUCCESS 1
#define UNSUCCESS 0
#define DUPLICATE -1
 Status SearchHash(HashTable H, KeyType K, int &p, int &c)
 {
    //在开放定址哈希表H中查找关键字为K的元素,若查找成功,以p指示待查数据
    //元素在表中位置,并返回SUCCESS；否则,以p指示插入位置,并返回UNSUCCESS,
    //c用以计算冲突次数,其初值为零,供建表插入时参考
    p = Hash(K);
    while (H.elem[p].key != NULLKEY && !EQ(K, H.elem[p].key))//该位置填有记录并且关键字不相等
        collision(p, ++c);//求得下一个探查地址p
    if (EQ(K, H.elem[p].key))
        return SUCCESS;//查找成功,p返回待查数据元素位置
    else
        return UNSUCCESS;//查找不成功,即H.elem[p].key == NULLKEY,p返回插入位置
}//SearchHash

Status InsertHash(HashTable &H, Elemtype e)
{
    //查找不成功时插入数据元素e到开放定址哈希表中,并返回OK；若冲突次数过大,则重建哈希表
    c = 0;
    if(SeatchHash(H, e.key, p, c)) {
        return DUPLICATE;
    } else if (c < hashsize[H,sizeindex]/2) {//表中已有与e有相同关键字的元素
        H.elem[p] = e; ++H.count; return OK;
    } else {
        RecreateHashTable(H); return UNSUCCESS;//重建哈希表
    }
}//InsertHash
{% endhighlight %}

#### 算法应用

一些可以考虑用哈希算法处理的算法问题,来[从头到尾彻底解析Hash 表算法](http://blog.csdn.net/v_JULY_v/article/details/6256463):

题目:假设目前有一千万个记录（这些查询串的重复度比较高，虽然总数是1千万，但如果除去重复后，不超过3百万个。一个查询串的重复度越高，说明查询它的用户越多，也就是越热门。），请你统计最热门的10个查询串，要求使用的内存不能超过1G。

思路：先用hash统计每个记录个数\(每次递增某条记录的重复数目时,基本为O\(1\)操作,比遍历快多了\)，再用堆求出top10.

题目：海量日志数据，提取出某日访问百度次数最多的那个IP。

思路：IP的数目还是有限的，最多2^32个，假设存储整个IP的话需要15个字节，那么所有IP将占用60G的空间，且每次都要在这60G中查找出相应的IP并计数。好的方案：将IP哈希到一张哈希表上，并且是3次哈希，第一次作为哈希表索引，其余两次作为校验。以后每次有IP访问就将其进行3次哈希，在哈希表中找到相应的值并递增访问次数。
