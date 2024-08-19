---
layout: next
title: Linux双网卡默认路由优先级不正确，导致网络不通问题定位
date: 2024-04-10 20:22:18
categories: troubleshooting
tags:
- troubleshooting
- Network
- Linux
---

## 问题描述
RHEL9 双网卡环境，两个网卡配置如下：(**eth0 走内网，eth1 走外网**)
```
eth0  192.168.10.20/24     网关: 192.168.10.254
eth1  10.206.216.92/24     网关: 10.206.216.254
```
配置完成后，`curl https://www.baidu.com`访问百度失败，发现网络不通。
## 定位步骤
<!-- more -->
使用`route`命令查路由表，发现eth0的默认路由Metric值为100，小于eth1的Metric(101)，这说明eth0默认路由优先级更高，**`curl https://www.baidu.com` 匹配的是eth0的默认路由，但我们需要通过eth1访问外网**，所以网络不通。
![](image1.png)
## 解决方法
有两种方法，任选其一即可
### 方法1: 手动修改eth0默认路由的metric值，需大于eth1
以下命令设置eth0的默认路由metric值为200，大于eth1的默认路由metric(101)
```bash
nmcli connection modify eth0 ipv4.route-metric 200
nmcli c up eth0
nmcli c reload
```
修改重启后仍有效 (配置写在`/etc/NetworkManager/system-connections/eth0.nmconnection`)
### 方法2：删除eth0的默认路由，只保留eth1的默认路由
通过配置`ipv4.never-default=no`，删除eth0网卡的默认路由，参考如下命令：
```bash
nmcli c mod eth0 ipv4.never-default no gw4 192.168.10.254
nmcli c up eth0
nmcli c reload
```
修改重启后仍有效 (配置写在`/etc/NetworkManager/system-connections/eth0.nmconnection`)
## 参考
[Red Hat Customer Portal -- Chapter 23. Managing the default gateway setting](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/managing-the-default-gateway-setting_configuring-and-managing-networking)

