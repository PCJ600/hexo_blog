---
layout: next
title: 如何查看Redis订阅的模式字符串
date: 2021-11-21 20:17:45
categories: Redis
tags: Redis
---

## 问题描述

`pubsub channels`可以查看Redis中被订阅的频道(**channel**)：

<!-- more -->
![](image1.png)


`pubsub numpat`可以查看被订阅的模式(**pattern**)数量：

```
# redis-cli pubsub numpat
(integer) 3
```

**问题：**

除了查看被订阅模式的数量，我还需要<font color = 'red'>**获取每个订阅模式字符串的内容**</font>，怎么做 ?



## 解决方法

google没搜到现成的命令，决定自己修改`redis-server`源码，打印模式链表的内容，用时3~5分钟，需要了解：

* Redis源码编译方法，参考[官网](https://github.com/redis/redis)或本人的[博客](https://blog.csdn.net/pcj_888/article/details/106483567)

* Redis服务器将所有模式的订阅信息保存在服务器状态的`pubsub_patterns`链表中。

#### 具体操作

1、下载Redis源码（这里用的是6.0.9版本的源码，[下载链接](https://github.com/redis/redis/releases/tag/6.0.9)），修改`pubsub.c`，自定义一个模式链表打印函数`myPubsubPatternsPrint`，实现参考如下：

```C
// pubsub.c
void myPubsubPatternsPrint()
{
    list *l = server.pubsub_patterns;
    listNode *cur = l->head;
    serverLog(LL_NOTICE, "[DEBUG] ALL PUBSUB PATTERNS: ");
    while (cur != NULL) {
        pubsubPattern *pp = (pubsubPattern *)(cur->value);
        robj *obj = pp->pattern;
        char *pattern = (char *)obj->ptr;
        serverLog(LL_NOTICE, "%s", pattern);
        cur = cur->next;
    }
}
```

2、重新编译并安装redis-server，后台gdb call一下自定义的打印函数`myPubsubPatternsPrint`得到模式链表的内容，如下图所示：
![](image2.png)

## 参考资料
【1】《Redis设计与实现》 第18章 发布与订阅
