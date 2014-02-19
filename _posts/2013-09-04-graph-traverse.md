---
layout: post
title: "图的深度优先搜索和广度优先搜索"
description: ""
category: 算法
tags: [数据结构]
---
{% include JB/setup %}

#### 深度优先搜索

{% highlight cpp linenos %}
Boolean visited[MAX];//访问标记数组

void DFSTraverse(Graph G, int v)
{
    for (v=0; v<G.vexnum; ++v) {
        visited[v] = FALSE;
    }
    for (v=0; v<G.vexnum; v++) {
        if (!visited[v])
            DFS(G, v);
    }
}

void DFS(Graph G, int v)
{
    visited[v] = TRUE;
    for (w = FirstAdjVex(G, v); w>=0; w = NextAdjVex(G, v))
        if (!visited[w])
            DFS(G, w);
}
{% endhighlight %}

<!--more-->
#### 广度优先搜索

{% highlight cpp linenos %}
void BFSTraverse(Graph G, int v)
{
    //使用辅助队列Q和访问标记数组visited
    for (v=0; v<G.vexnum; v++) {
        visited[v] = FALSE;
    }
    InitQueue(Q);
    for (v=0; v<G.vexnum; v++) {
        if (!visited[v]) {
            visited[v] = TRUE;
            EnQueue(Q, v);//入队
            while (!QueueEmpty(Q)) {
                DeQueue(Q, u);//出队
                for (w = FirstAdjVex(G, u); w>=0; w = NextAdjVex(G, u))
                    if (!visited[w]) {
                        visited[w] = TRUE;
                        EnQueue(Q, w);
                    }//if
            }//while
        }//if
    }//for
}
{% endhighlight %}