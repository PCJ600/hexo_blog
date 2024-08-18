---
layout: next
title: Redis抓包分析案例
date: 2021-11-06 20:14:29
categories:
- troubleshooting
- Redis
tags: 
- troubleshooting
- Redis
---

## 问题描述

**业务背景：** 有一台由多个Docker容器组成的仿真设备环境，1个Docker部署Redis服务端，剩余每个Docker都作为Redis客户端，用于模拟一块单板。

**问题：** 仿真设备启动中，**概率出现Redis的某个字符串键被不符合预期地改写成空串**，导致客户端Docker从Redis获取的数据有问题。**需要定位是哪个Docker上的哪个进程改写了Redis数据库。**

<!-- more -->

## 定位思路

* 先尝试排查客户端代码。但此案例中涉及业务代码过多(200w+行代码)，且Redis客户端的形式也很多（有`goredis`, `redis-py`, `hiredis`, `redis-cli`等）。看来肉眼扫描代码的笨方法不靠谱，而且这种**寄希望于碰运气蒙混过关**的思考方式实在不像是一个程序员。

那么既然排查客户端有困难，能不能换个角度思考，比如说从服务端入手？以下给出第二种思路：

* **通过在服务端抓包，得到所有对这个字符串键做set操作的报文，再根据报文中的IP和端口号得到进程ID**。听起来可行，下面举一个简化的案例介绍具体的操作方法。

## 具体方法

利用tcpdump，在Redis服务端后台抓取对这个字符串键做set操作的报文，再根据报文中的IP和端口号确认进程即可。

如下图所示，使用命令`tcpdump -i any port 6379 | grep set | grep [key]` ，得到客户端IP是localhost,端口号是44880。 

![](image1.png)


再根据报文中的IP和端口号确认进程号，可以用`netstat`, `ss`, `lsof`等实现。以下仅给出`netstat`方式，命令：`netstat -anltp | grep 端口号`，得到进程ID为6748，如下图所示：
![](image2.png)


#### 思考：

如果我只需要抓取将某个特定字符串键（比如"hello"）写成**某个特定值**（比如写成空串）的报文，怎么做？

除了`grep`，下面再给出一种方法，利用**tcpdump的根据报文特征过滤**的技巧：

1、使用tcpdump的-X选项查看报文详细内容，重点看Redis请求在TCP报文中是如何存储的：
![](image3.png)


可以看出, `set hello "world"`命令在报文中存储的协议内容如下：

```
*3 $3 set $5 hello $5 world
```

协议内容的解析参考[Redis协议规范（RESP）](https://redis.io/topics/protocol) ， 以下只做简单的解释：

* *3 表示数组长度为3， 数组元素依次为 ["set" "hello" "world"]
* $3 表示字符串 "set" 的长度
* 第一个 $5 表示字符串 "hello" 的长度。
* 第二个 $5 表示字符串 "world" 的长度。

2、再根据TCP报文内容的特征过滤。此案例中，筛选条件可以是同时匹配 “set” "hello" "$0"(**0匹配键值的长度，用来匹配空串**），可以使用命令`tcpdump -i any port 6379 -X and 'ip[60:2] == 0x7365' and 'ip[69:4] == 0x68656c6c' and 'ip[76:2] == 0x2430'` 抓取所有将字符串键"hello"写为空串的报文，参考下图：
![](image4.png)
## 参考资料

【1】[Redis协议规范（RESP）](https://redis.io/topics/protocol)

【2】[网络基本功（十八）：细说tcpdump的妙用（下）](https://wizardforcel.gitbooks.io/network-basic/content/17.html)
