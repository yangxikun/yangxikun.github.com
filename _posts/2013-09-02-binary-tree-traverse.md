---
layout: post
title: "二叉树的遍历"
description: ""
category: 算法
tags: [数据结构]
---
{% include JB/setup %}

###存储结构

>链式存储结构

{% highlight cpp linenos %}
typedef struct BiTNode{
    TElemType data;
    struct BiTNode *lchild,*rchild;//左右孩子指针
}BiTNode,*BiTree;
{% endhighlight %}

###先序遍历(根左右)

{% highlight cpp linenos %}
Status PreOrderTraverse( BiTree T ){
    InitStack( S );p=T;
    while( p || !StackEmpty( S ) ){
        if( p ){
            visite( p );//访问p
            StackPush( S, p );
            p = p->lchild;
        }else{
            StackPop( S, p );
            p = p->rchild;
        }
    }//while
    return OK;
}
{% endhighlight %}

###中序遍历(左根右)
{% highlight cpp linenos %}
Status InOrderTraverse( BiTree T ){
    InitStack( S );p=T;
    while( p || !StackEmpty( S ) ){
        if( p ){
            StackPush( S, p );
            p = p->lchild;
        }else{
            StackPop( S, p );
            visite( p );
            p = p->rchild;
        }
    }//while
}
{% endhighlight %}

###后序遍历(左右根)
{% highlight cpp linenos %}
Status PostOrderTraverse( BiTree T ){
    InitStack( S );p=T;
    while( p || !StackEmpty( S ) ){
        while( p ){
            StackPush( S, p );
            p = p->lchild;
        }//while
        GetTop( S, p );//取得栈顶元素的值
        if( p->rchild == NULL || lastVisite == p->rchild ){
            visite( p );
            lastVisite = p;//记录上一次访问的结点
            StackPop( S, p );
            p = NULL;
        }else{
            p = p->rchild;
        }
    }//while
    return OK;
}
{% endhighlight %}