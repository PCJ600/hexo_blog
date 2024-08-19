---
layout: next
title: 使用arping检测IP地址是否冲突
date: 2024-07-04 22:05:35
categories: Linux
tags: Linux
---

## arping简介
在Linux中，`arping`是一个用来发送ARP请求到一个相邻主机的工具，通常用于检测网络上的IP地址冲突。

## 使用arping检测IP地址是否冲突的方法

例1：使用如下命令检测10.206.216.95是否冲突 (使用-I参数指定网络接口)
```
# arping -I eth0 10.206.216.95
ARPING 10.206.216.95 from 10.206.216.91 eth0
Unicast reply from 10.206.216.95 [00:50:56:89:A7:3E]  0.627ms
Unicast reply from 10.206.216.95 [00:50:56:89:A7:3E]  0.683ms
Unicast reply from 10.206.216.95 [00:50:56:89:A7:3E]  0.718ms
```
查看输出，发现收到了ARP应答，说明IP有冲突，且对应的MAC地址为00:50:56:89:A7:3E

例2：使用如下命令检测10.206.216.92是否冲突
```
# arping 10.206.216.92
ARPING 10.206.216.92 from 10.206.216.91 eth0
```
查看输出，发现未收到任何ARP应答，说明IP没有冲突

## 参考
[https://man7.org/linux/man-pages/man8/arping.8.html](https://man7.org/linux/man-pages/man8/arping.8.html)
