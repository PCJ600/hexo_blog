---
layout: next
title: Hiredis查询失败时出现key丢失问题定位
date: 2021-11-30 20:25:25
categories:
- troubleshooting
tags:
- Redis
- troubleshooting
---

### 问题描述

hiredis查询key失败后，出现key丢失的问题。

REDIS版本及源码：[6.0.10](https://github.com/redis/redis/releases/tag/6.0.10)

hiredis版本及源码：[v1.0.2](https://github.com/redis/hiredis/releases/tag/v1.0.2)

<!-- more -->

**案例描述：**

REDIS中预先写入1个字符串键"hello"，客户端代码基于hiredis，创建100个读线程和100个写线程，每个线程里发起一次短连接读写key，代码参考：

```C
#include <assert.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <time.h>
#include <hiredis/hiredis.h>
#include <pthread.h>

#define SERV_IP "127.0.0.1"
#define SERV_PORT 6379
#define NUM_READER 100
#define NUM_WRITER 100

static const int OK = 0;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

int DBGetString(const char *key, char *value) {
    struct timeval tv;
    tv.tv_sec = 5; tv.tv_usec = 0;
    redisContext *c = redisConnectWithTimeout(SERV_IP, SERV_PORT, tv);
    assert(c != NULL && !c->err);

    char cmd[256];
    sprintf(cmd, "GET %s", key);
    redisReply *reply = (redisReply *)redisCommand(c, cmd);
    assert(reply->str != NULL);
    strcpy(value, reply->str);
    freeReplyObject(reply);
    redisFree(c);
    return OK;
}

int DBSetString(const char *key, const char *value) {
    struct timeval tv;
    tv.tv_sec = 5; tv.tv_usec = 0;
    redisContext *c = redisConnectWithTimeout(SERV_IP, SERV_PORT, tv);
    assert(c != NULL && !c->err);

    char cmd[256];
    sprintf(cmd, "SET %s %s", key, value);
    redisReply *reply = (redisReply *)redisCommand(c, cmd);
    freeReplyObject(reply);
    redisFree(c);
    return OK;
}

void *Reader(void *args) {
    char buf[256];
    DBGetString("hello", buf);
    printf("%s\n", buf);
    return NULL;
}

void *Writer(void *args) {
    DBSetString("hello", "world");
    return NULL;
}

void testcase() {
    time_t t1 = time(NULL);
    pthread_t wid[NUM_WRITER];
    pthread_t rid[NUM_READER];

    for (int i = 0; i < NUM_WRITER; ++i) {
        pthread_create(&wid[i], NULL, Writer, NULL);
    }
    for (int i = 0; i < NUM_READER; ++i) {
        pthread_create(&rid[i], NULL, Reader, NULL);
    }

    for (int i = 0; i < NUM_WRITER; ++i) {
        pthread_join(wid[i], NULL);
    }
    for (int i = 0; i < NUM_READER; ++i) {
        pthread_join(rid[i], NULL);
    }
    time_t t2 = time(NULL);
    printf("%ld s\n", t2 - t1);
}

int main() {
    testcase();
    return 0;
}
```



### 定位思路

Redis key丢失一般有如下原因：

* key被客户端删除。
* key是过期键，到期自动删除。
* 因为内存不足导致key被逐出。

本案例中的客户端只有get,set，没有删除操作，未设置键的过期时间，且机器内存也足够（4G以上）。初步分析不出原因，只能结合REDIS源码正向定位，于是有了第二种思路：

***REDIS是基于内存的数据库，字符串键肯定在REDIS-SERVER进程中的某个内存地址处，key丢失时这个地址的内容肯定被改写，这时只需要GDB打个数据断点，看下调用栈即可分析原因。***

以下给出详细的定位过程。

### 定位过程

1、使用GCC的-g -O0编译选项，重新编译redis-server, hiredis源代码。（这一步非必要，目的是更方便调试）

2、GDB打数据断点
![](image1.png)

3、断点触发，观察调用栈
![](image2.png)

观察第5帧，`freeMemoryIfNeeded`，说明触发了REDIS的内存淘汰机制，GDB打印出的REDIS配置项`maxmemory`仅为1048576字节(表示内存大小为1M)，且内存淘汰机制为`allkeys-lru`，这个机制会导致key被回收。



### 解决方法

修改redis配置项maxmemory，内存给多点。（这里给1073741824，表示1G内存）




