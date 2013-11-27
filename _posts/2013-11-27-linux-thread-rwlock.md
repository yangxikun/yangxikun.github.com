---
layout: post
title: "linux线程使用读写锁共享全局变量"
description: ""
category: Linux
tags: [线程]
---
{% include JB/setup %}

*学习自：[Linux线程同步(3): 读写锁(rwlock)](http://blog.csdn.net/dai_weitao/article/details/1752843)和[linux网络编程之posix 线程（一）：pthread 系列函数 和 简单多线程服务器端程序](http://blog.csdn.net/jnu_simba/article/details/9106513)*

#### 读写锁 (rwlock)功能特点简介

读写锁实际是一种特殊的自旋锁，它把对共享资源的访问者划分成读者和写者，读者只对共享资源进行读访问，写者则需要对共享资源进行写操作。这种锁相对于自旋锁而言，能提高并发性，因为在多处理器系统中，它允许同时有多个读者来访问共享资源，最大可能的读者数为实际的逻辑CPU数。写者是排他性的，一个读写锁同时只能有一个写者或多个读者（与CPU数相关），但不能同时既有读者又有写者。

在读写锁保持期间也是抢占失效的。

如果读写锁当前没有读者，也没有写者，那么写者可以立刻获得读写锁，否则它必须自旋在那里，直到没有任何写者或读者。如果读写锁没有写者，那么读者可以立即获得该读写锁，否则读者必须自旋在那里，直到写者释放该读写锁。

1. 特性：

一次只有一个线程可以占有写模式的读写锁, 但是可以有多个线程同时占有读模式的读写锁. 正是因为这个特性,

当读写锁是写加锁状态时, 在这个锁被解锁之前, 所有试图对这个锁加锁的线程都会被阻塞.

当读写锁在读加锁状态时, 所有试图以读模式对它进行加锁的线程都可以得到访问权, 但是如果线程希望以写模式对此锁进行加锁, 它必须直到知道所有的线程释放锁.

通常, 当读写锁处于读模式锁住状态时, 如果有另外线程试图以写模式加锁, 读写锁通常会阻塞随后的读模式锁请求, 这样可以避免读模式锁长期占用, 而等待的写模式锁请求长期阻塞.

2. 适用性:

读写锁适合于对数据结构的读次数比写次数多得多的情况. 因为, 读模式锁定时可以共享, 以写模式锁住时意味着独占, 所以读写锁又叫共享-独占锁.

#### 实例：使用读写锁实现线程间共享全局变量

{% highlight cpp linenos %}
#include<pthread.h>
#include<stdio.h>
#include<unistd.h>
#include<errno.h>
#include<stdlib.h>
#include<string.h>

/* 读写锁对象 */
static pthread_rwlock_t rwlock;

int a = 0;

void* thread_function (void *arg) {

    int res;
    /* 获取写锁 */
    res = pthread_rwlock_wrlock(&rwlock);
    if (res != 0) {
        printf("%s\n", strerror(res));
        exit(EXIT_FAILURE);
    }

    printf("子线程获取写锁\n");
    printf("原来a的值:%d\n", a);
    a = 100;
    printf("修改后a的值:%d\n", a);

    /* 释放写锁 */
    res = pthread_rwlock_unlock(&rwlock);
    if (res != 0) {
        printf("%s\n", strerror(res));
        exit(EXIT_FAILURE);
    }
    printf("子线程释放写锁\n");

    return NULL;
}

int main() {

    int res;
    /* 初始化写写锁 */
    res = pthread_rwlock_init(&rwlock, NULL);
    if (res != 0) {
        perror("pthread_rwlock_init");
        exit(EXIT_FAILURE);
    }

    /* 创建新线程 */
    pthread_t thread;
    res = pthread_create(&thread, NULL, &thread_function, NULL);
    if (res != 0) {
        printf("%s\n", strerror(res));
        exit(EXIT_FAILURE);
    }

    /* 获取写锁 */
    res = pthread_rwlock_wrlock(&rwlock);
    if (res != 0) {
        printf("%s\n", strerror(res));
        exit(EXIT_FAILURE);
    }

    printf("主线程获取写锁\n");
    printf("原来a的值:%d\n", a);
    a = 10;
    printf("修改后a的值:%d\n", a);
    printf("主线程睡眠5s\n");
    sleep(5);

    /* 释放写锁 */
    res = pthread_rwlock_unlock(&rwlock);
    if (res != 0) {
        printf("%s\n", strerror(res));
        exit(EXIT_FAILURE);
    }
    printf("主线程释放写锁\n");

    pthread_join(thread, NULL);
    return 0;
}
{% endhighlight %}

