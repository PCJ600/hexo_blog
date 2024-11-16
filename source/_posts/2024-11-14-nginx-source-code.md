---
layout: next
title: Nginx源码解读
date: 2024-11-14 19:58:46
categories: draft
tags: draft
---
https://www.bilibili.com/video/BV1oi421m7Qa?spm_id_from=333.788.videopod.episodes&vd_source=d8559c2d87607be86810cd806158bb86
https://102no.com/2021/06/07/10-nginx-learn-books/#%E3%80%8ANginx%E5%BC%80%E5%8F%91%E4%BB%8E%E5%85%A5%E9%97%A8%E5%88%B0%E7%B2%BE%E9%80%9A%E3%80%8B
(Nginx/Redis/kernel)

Nginx 1.20.1
* Nginx.conf如何解析
* 多进程网络连接
* 内存池实现
* 线程池源码
* 进程间通信共享内存的实现

经典问题:
* 多进程如何实现
* 惊群
* 内存池
* 多个进程锁

# ngx_modules.c文件定义ngx_modules数组

# Nginx.conf解析
ngx_set_worker_processes


单独用len计长度, 便于内存池管理
```
typedef struct {
    size_t      len;
    u_char     *data;
} ngx_str_t;
```

多个进程监听不同的端口怎么做的?

ngx_http_optimize_servers
ngx_http_init_listening
ngx_http_add_listening
ngx_create_listening

# 内存池
Nginx用在哪里?
连接建立, 为fd分配内存池


内存池实现ngx_calloc.c
```
typedef struct ngx_pool_large_s  ngx_pool_large_t;

struct ngx_pool_large_s {
    ngx_pool_large_t     *next;
    void                 *alloc;
};

单链表
(alloc | next) -> (alloc | next) -> (alloc | next)

typedef struct {
    u_char               *last;				// 当前内存可分配的地址
    u_char               *end;				// 当前内存结尾地址
    ngx_pool_t           *next;				// 指向下一块内存的结构体
    ngx_uint_t            failed;			// 记录分配失败次数
} ngx_pool_data_t;

typedef struct {
    u_char               *last;
    u_char               *end;
    ngx_pool_t           *next;
    ngx_uint_t            failed;
} ngx_pool_data_t;

struct ngx_pool_s {
    ngx_pool_data_t       d;
    size_t                max;
    ngx_pool_t           *current;
    ngx_chain_t          *chain;
    ngx_pool_large_t     *large;
    ngx_pool_cleanup_t   *cleanup;
    ngx_log_t            *log;
};
```


## 问题: C10K问题如何解决的
C10K问题: 如何在一台物理机支持1万个并发连接
IO 多路复用(epoll)


问题: Master和Worker进程分别是什么,
master 加载配置文件，管理子进程
worker 每个worker进程可同时处理多个请求

问题: 多个worker进程如何监听同一个端口的, 哪一个worker处理请求(惊群) ?
fork
每个进程epoll_wait -> accept， 但只有一个进程accept执行成功

只让一个进程的fd加到epoll里
atomic lock; (共享内存)


问题: 多个server怎么启动的，端口LISTEN在哪?