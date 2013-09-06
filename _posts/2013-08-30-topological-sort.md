---
layout: post
title: "拓扑排序"
description: ""
category: 算法
tags: [数据结构]
---
{% include JB/setup %}

###什么是拓扑序列

>由某个集合上的一个偏序得到该集合上的一个全序，这个操作称之为拓扑排序。

>若集合X上的关系是R,且R是自反的、反对称的和传递的，则称R是集合X上的偏序关系。

>>偏序关系又称半序关系。

>>设A是一个非空集，P是A上的一个关系，若关系P是自反的、反对称的、和传递的，则称P是集合A上的偏序关系。
即P适合下列条件：

>>1.对任意的a∈A,\(a,a\)∈P;

>>2.若\(a,b\)∈P且\(b,a\)∈P,则a=b;

>>3.若\(a,b\)∈P,\(b,c\)∈P,则\(a,c\)∈P,则称P是A上的一个偏序关系。带偏序关系的集合A称为偏序集或半序集。

>设R是集合X上的偏序\(Partial Order\),如果对每个x,y属于X必有xRy 或 yRx，则称R是集合X上的全序关系。

>比较简单的理解：偏序是指集合中只有部分成员可以比较，全序是指集合中所有的成员之间均可以比较。

###AOV网

>用顶点表示活动，用弧表示活动间的优先关系的有向图称为顶点表示活动的网（Activity On Vertex Network)，简称AOV网。

>AOV网是一种有向无回路的图。

###拓扑排序的使用

>验证一个有向图无回路存在:

>>1.在有向图中选一个没有前驱的顶点且输出之.

>>2.从图中删除该顶点和所有以它为尾的弧.

>>3.设一栈暂存所有入度为零的顶点.

{% highlight cpp linenos %}
Status TopologicalSort(ALGraph G){
    //有向图G采用邻接表存储结构
    //若G无回路,则输出G的顶点的一个拓扑序列并返回OK,否则ERROR.
    FindInDegree( G, indegree );    //对各顶点求入度indegree[0...vernum-1]
    InitStack( S );
    for( i=0; i<G.vexnum; ++i ) //建零入度顶点栈S
        if( !indegree[i] ) StackPush( S, i ); //入度为0者进栈
    /*上面步骤可以在FindInDegree()函数中完成*/
    count = 0 ; //对输出顶点计数
    while( !StackEmpty( S ) ){
        StackPop( S, i );printf( i, G.vertices[i].data );++count;   //输出i号顶点并计数
        for( p = G.vertices[i].firstarc; p; p=p->nextarc ){//对i号顶点的每个邻接点的入度减1
            if( !( --indegree[p->adjvex] ) ) StackPush( S, p->adjvex );//若入度为零,则入栈
        }//for
    }//while
    if( count < G.vexnum ) return ERROR;    //该有向图有回路
    else return OK;
}//TopologicalSort
{% endhighlight %}