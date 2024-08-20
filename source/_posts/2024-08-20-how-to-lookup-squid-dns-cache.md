---
layout: next
title: 如何查看Squid的DNS缓存
date: 2024-08-20 19:48:46
categories: Squid
tags: Squid
---

使用`squidclient mgr:ipcache`命令查看Squid的DNS缓存记录
如果squid端口不是3128, 需要指定端口号, `squidclient -p {port} mgr:ipcache`

<!-- more -->

```
# squidclient mgr:ipcache
...
IP Cache Statistics:
...
IP Cache Contents:
 Hostname                        Flg lstref    TTL  N(b)
 www.trendmicro.com                      19     41  1( 0)                                 23.195.109.63-OK
 localhost6.localdomain6          H      93     -1  1( 0)                                           ::1-OK
 localhost6                       H      93     -1  1( 0)                                           ::1-OK
 localhost.localdomain            H      93     -1  1( 0)                                           ::1-OK
 localhost                        H      93     -1  1( 0)                                           ::1-OK
 localhost4.localdomain4          H      93     -1  1( 0)                                     127.0.0.1-OK
 localhost4                       H      93     -1  1( 0)                                     127.0.0.1-OK
 www.baidu.com                         1139  -1079  2( 0)                                103.235.47.188-OK
                                                                                          103.235.46.96-OK
```

## DNS请求抓包
```
tcpdump udp port 53 -w dns.pcap
curl -x 127.0.0.1:3128 https://www.baidu.com
```
10.206.216.93为nameserver的IP
![](image1.png)
## 参考
[https://wiki.squid-cache.org/Features/CacheManager/IpCache](https://wiki.squid-cache.org/Features/CacheManager/IpCache)

