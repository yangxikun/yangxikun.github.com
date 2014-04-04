---
layout: post
title: "最小生成树(普利姆算法)"
description: ""
category: 算法
tags: [数据结构]
---
{% include JB/setup %}

#### 思想


1. 假设最小生成树点集为V,图的所有点集为U,初始化V含有U中任意一个顶点；
2. 从U-V中选出一个离V最近的一个顶点加入V中；
3. 更新U-V中顶点到V的最短路径；
4. U不为空转步骤2,为空继续；
5. 结束。

<!--more-->

#### 辅助


1. closedge\[\]数组记录了相应顶点到V的最短距离`struct{ VertexType adjvex; VRType lowcost;}closedge[MAX]`；
2. 图的存储结构为邻接矩阵。

#### 算法
{% highlight cpp linenos %}
void MiniSpanTree_PRIM(MGraph G, VertexType u)
{
    //MGraph G 图的邻接矩阵
    k = LocateVex(G, u);
    for (j=0; j<G.vexnum; j++)//辅助数组初始化
        if(j != k)
            closedge[j] = {u, G.arcs[k][j].adj};//{adjvex, lowcost}
    closedge[k] = {u, 0};
    for (j=0; j<G.vexnum; j++) {
        k = min(closedge);//从记录数组中选取最小的
        closedge[k].lowcost = 0;//将k顶点并入V集
        for (j=0; j<G.vexnum; j++)/*如果把辅助数组改为链表的话,对于G.vexnum很大的情况好点,每次把并入V集合的节点free掉*/
            if(G.arcs[k][j].adj < closedge[j].lowcost)
                closedge[j] = {G.vexs[k], G.arcs[k][j].adj};
    }//for
}
{% endhighlight %}