---
layout: post
title: "AOE网(关键路径)"
description: ""
category: 算法
tags: [数据结构]
---
{% include JB/setup %}

#### 描述

AOE网是一个带权的有向无环图,其中,顶点表示事件,弧表示活动,权表示活动持续的时间.通常,AOE网可用来估算工程的完成时间.

网中只有一个入度为零的顶点[源点]和一个出度为零的顶点[汇点].

![AOE](/assets/img/201309070101.jpg)

<!--more-->

#### 问题

1. 完成整项工程至少需要多少时间?
2. 哪些活动是影响工程进度的关键?

#### 思想

1. AOE网中有些活动是可以并行地运行,所以完成工程的最短时间是从开始点到完成点的最长路径的长度.
2. 路径长度最长的路径叫做关键路径.
3. 假设开始点是v1,从v1到vi的最长路径叫做事件vi的最早发生时间[因为vi事件要等到所有以vi为头的弧[活动]完成才能发生].这个时间决定了所有以vi为尾的弧所表示的活动的最早发生时间.
4. 用`e[i]`表示活动ai的最早开始时间,用`l[i]`表示活动的最迟开始时间[在不推迟整个工程的前提下,活动ai的最迟开始时间]，两者之差`l[i]-e[i]`意味着完成活动ai的时间剩余量.我们把`l[i]=e[i]`的活动叫做关键活动.
5. 关键路径上的所有活动都是关键活动,因此提前完成非关键活动并不能加快工程进度.

#### 求解分析

辨别关键活动就是要找`e[i]=l[i]`的活动.设AOE网中活动的最早开始时间为`e[i]`和最迟开始时间为`l[i]`，事件的最早发生时间为`ve[j]`和最迟发生时间为`vl[j]`。如果活动ai由弧`j,k`表示，其持续时间记为`dut[j,k]`,则有如下关系:

`e[i]=ve[j]`

`l[i]=vl[k]-dut[j,k]`

求解`ve[j]`和`vl[j]`如下:

1. 从`ve[0]=0`开始向前递推：`ve[j]=Max{ve[i]+dut[i,j]}`其中`i,j`属于T,T是所有以第j个顶点为头的弧的集合

2. 从`vl[n-1]=ve[n-1]`起向后递推:`vl[i]=Min{vl[j]-dut[i,j]}`其中`i,j`属于S,S是所有以第i个顶点为尾的弧的集合

#### 算法流程

1. 输入e条弧,建立AOE网为邻接表存储结构；
2. 从源点出发,令ve[0]=0,按拓扑有序求其余各顶点的最早发生时间；
3. 从汇点出发,令vl[n-1]=ve[n-1],按逆拓扑有序求其余顶点的最迟开始时间；
4. 根据各顶点的ve和vl,求每条弧s的最早开始时间e(s)和最迟开始时间l(s),若某条弧满足e(s)=l(s),则为关键活动。

#### 辅助

栈T用于存储拓扑序列；

ve数组记录事件最早开始时间,vl数组记录事件最迟开始时间。

#### 代码

{% highlight cpp linenos %}
Status TopologicalSort(ALGraph G, Stack &T){
    //有向图G采用邻接表存储结构
    //若G无回路,则输出G的顶点的一个拓扑序列并返回OK,否则ERROR.
    FindInDegree( G, indegree );    //对各顶点求入度indegree[0...vernum-1]
    InitStack( S );
    InitStack( T );
    ve[0..G.vexnum-1] = 0;
    for( i=0; i<G.vexnum; ++i ) //建零入度顶点栈S
        if( !indegree[i] ) StackPush( S, i ); //入度为0者进栈
    /*上面步骤可以在FindInDegree()函数中完成*/
    count = 0 ; //对输出顶点计数
    while( !StackEmpty( S ) ){
        StackPop( S, i );StackPush( T, i );++count;   //i号顶点入栈T并计数
        for( p = G.vertices[i].firstarc; p; p = p->nextarc ){//对i号顶点的每个邻接点的入度减1
            if( !( --indegree[p->adjvex] ) ) StackPush( S, p->adjvex );//若入度为零,则入栈
            if( ve[i]+*(p->info) > ve[p->adjvex] ) ve[k] = ve[i]+*(p->info);//相当于ve(j)=Max{ve(i)+dut(<i,j>)}
        }//for
    }//while
    if( count < G.vexnum ) return ERROR;    //该有向图有回路
    else return OK;
}//TopologicalSort

Status CriticalPath(ALGraph G){
    //G为有向网,输出G的各项关键活动
    if( !TopologicalOrder(G, T) ) return ERROR;
    vl[0..G.vexnum-1] = ve[0..G.vexnum-1];//初始化顶点事件的最迟发生时间
    while( !StackEmpty(T) ) //按逆拓扑序列求各顶点的vl值
        for( StackPop( T, j ), p = G.vertices[j].firstarc; p; p = p->nextarc ){
            k = p->adjvex; dut = *(p->info);
            if( vl[k]-dut < vl[j] ) vl[j] = vl[k]-dut;
        }//for
    for( j=0; j<G.vexnum; ++j )//求每条边的最早开始时间和最迟开始时间
        for( p = G.vertices[j].firstarc; p; p = p->nextarc ){
            k = p->adjvex; dut = *(p->info);
            ee = ve[j]; el = vl[k]-dut;
            tag = (ee == el) ? 1 : 0;
            printf( j, k, dut, ee, el, tag);//输出关键活动
        }//for p
}//CriticalPath
{% endhighlight %}