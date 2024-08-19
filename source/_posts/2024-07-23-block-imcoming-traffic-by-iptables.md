---
layout: next
title: 自定义iptables链，阻止所有进入流量的方法
date: 2024-07-23 22:27:33
categories: iptables
tags: iptables
---

## 需求描述
客户的Linux主机出现网络安全事件时, 需要先帮客户止血，再分析定位。

打开止血功能后，需要阻止所有进入流量(22号端口除外，系统内部流量和出去流量不受影响)；关闭止血功能后，解除限制，且原有的iptables规则要保持不变，如何实现？

## 实现方法
### 阻止所有进入流量的方法(22号端口除外)
1. 先自定义一个链BLOCK_IN, 新增规则: 22号端口ACCEPT, 其余端口DROP, 如下：
```bash
iptables -N BLOCK_IN 							 # 创建一个自定义链，名字是BLOCK_IN
iptables -A BLOCK_IN -p tcp --dport 22 -j ACCEPT # 允许22号端口通过
iptables -A BLOCK_IN -j DROP 					 # 丢弃
```

<!-- more -->

2. 再从INPUT链的首部新增一条规则：从网卡eth0进来的流量跳转到自定义链BLOCK_IN（注意这里把eth0替换成你机器的网卡名），如下:
```bash
iptables -I INPUT -i eth0 -j BLOCK_IN
```
在另一台机器上测试端口，发现超时，再测试SSH可以连接成功，说明修改生效。 使用`iptables -L BLOCK_IN -v -x`查看接收和丢弃的流量计数。
### 解除限制，放开进入流量
只需删除这个自定义链即可，原有的规则不受影响。方法如下：
```bash
iptables -D INPUT -i eth0 -j BLOCK_IN	# 删除之前创建的规则，这一步不做后面删除链的时候会失败
iptables -F BLOCK_IN 					# 清空BLOCK_IN链的规则
iptables -X BLOCK_IN 					# 删除整个BLOCK_IN链
```
