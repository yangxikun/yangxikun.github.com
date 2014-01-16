---
layout: post
title: "C语言使用JSON，cJSON库的使用"
description: ""
category: C
tags: [C库]
---
{% include JB/setup %}

学习了点socket编程，发现通过TCP/UDP进行通信接收到的是连续的字节内容，于是想要传输有结构的数据，就想到了JSON，上网查了下，找到了从[cJSON库](http://sourceforge.net/projects/cjson/)，这个库简单易用，很不错。

<!--more-->
结构体：
{% highlight cpp linenos %}
typedef struct cJSON {
    struct cJSON *next,*prev;   /* next/prev allow you to walk array/object chains. Alternatively, use GetArraySize/GetArrayItem/GetObjectItem */
    struct cJSON *child;        /* An array or object item will have a child pointer pointing to a chain of the items in the array/object. */
 
    int type;                   /* The type of the item, as above. */
 
    char *valuestring;          /* The item's string, if type==cJSON_String */
    int valueint;               /* The item's number, if type==cJSON_Number */
    double valuedouble;         /* The item's number, if type==cJSON_Number */
 
    char *string;               /* The item's name string, if this item is the child of, or is in the list of subitems of an object. */
} cJSON;
{% endhighlight %}

示例：
{% highlight cpp linenos %}
#include<stdio.h>
#include<string.h>
#include "cJSON/cJSON.h"
#include "cJSON/cJSON.c"

int main() {
    char *json = "{\"name\":\"me\"}";
    cJSON *name = cJSON_Parse(json);
    printf("%s\n", cJSON_GetObjectItem(name, "name")->valuestring);
    
    cJSON *user_list = cJSON_CreateObject();
    cJSON *u = cJSON_CreateObject();
    cJSON_AddStringToObject(u, "username", "kun");
    cJSON_AddItemToObject(user_list, "kun",u);
    u = cJSON_CreateObject();
    cJSON_AddStringToObject(u, "username", "liang");
    cJSON_AddItemToObject(user_list, "liang", u);

    int length = cJSON_GetArraySize(user_list);
    printf("%d\n", length);
    int i;
    for (i=0; i<length; i++) {
        cJSON *item = cJSON_GetArrayItem(user_list, i);
        printf("%s\n", cJSON_GetObjectItem(item, "username")->valuestring);
    }
    printf("%s\n", cJSON_PrintUnformatted(user_list));

    cJSON_Delete(name);
    cJSON_Delete(user_list);

    return 0;
}
{% endhighlight %}

输出：<br />
![demo](/assets/img/201312210101.png)
