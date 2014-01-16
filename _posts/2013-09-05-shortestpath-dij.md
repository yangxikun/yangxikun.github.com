---
layout: post
title: "最短路径问题(迪杰斯特拉算法)"
description: ""
category: 算法
tags: [数据结构]
---
{% include JB/setup %}

###思想
1.Dijkstra求有向网G的某一个顶点到其余顶点的最短路径问题.
2.假设V为已求得最短路径的点的集合,U为所有顶点的集合.
3.从U-V中选出一个离起点最近的顶点加入V中.
4.更新U-V中的顶点到起点的最短距离.
5.如果U不为空,转步骤3,否则继续.
6.结束

<!--more-->
###辅助
1.路径P\(二维数组boolean型\),当为true时,表示P\[i\]\[j\]从起点到顶点i的最短路径经过顶点j,为false,表明没有经过顶点j.
2.路径权值D\(一维数组\),D[i]的值为从起点到顶点i的最短路径权值.
3.标记数组final\(一维数组boolean型\),final[i]为true时,表示顶点i加入了V集合.

###算法
{% highlight cpp linenos %}
void ShortestPath_DIJ( MGraph G, int v0, PathMatrix &P, ShortPathTable &D){
    for( v=0; v<G.vexnum; v++ ){
        final[v] = FALSE;//初始化标记数组
        D[v] = G.arcs[v0][v];//初始化最短路径
        for( w=0; w<G.vexnum; w++ ) P[v][w] = FALSE;//初始化路径
        if( D[v] < MAX ){//如果v0到v有路径
            P[v][v0] = TRUE;
            P[v][v] = TRUE;
        }//if
    }//for
    D[v0] = 0;final[v0] = TRUE;//初始化V集合,含有v0起点
    for( i=1; i<G.vexnum; i++){
        min = MAX;//当前所知距离v0顶点的最近距离
        for( w=0; w<G.vexnum; w++ )//寻找满足要求的顶点
            if( !final[w] && D[w]<min ){
                v =w;
                min = D[w];
            }//if
        final[v] = TRUE;//加入V集合
        for( w=0; w<G.vexnum; w++ )
            if( !final[w] && (min+G.arcs[v][w]<D[w]) ){//更新U-V中顶点到起点的最短路径
                D[w] = min + G.arcs[v][w];
                P[w][1...G.vexnum-1] = P[v][1...G.vexnum-1];
                P[w][w] = TRUE;
            }//if
    }//for
}
{% endhighlight %}