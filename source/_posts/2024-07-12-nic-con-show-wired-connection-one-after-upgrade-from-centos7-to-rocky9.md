---
layout: next
title: 从Centos7升级到Rocky linux 9后，网卡连接显示Wired connection 1问题定位
date: 2024-07-12 22:11:04
categories: troubleshooting
tags:
- Linux
- troubleshooting
---


## 问题描述
从Centos7升级到Rocky9后, 发现网卡eth0的IP不正确。通过`nmcli`查看网卡连接，找不到name为eth0的连接，只显示'Wired connection 1'

<!-- more -->

![](image1.png)
查看`/etc/NetworkManager/system-connections/`，发现找不到网卡配置文件。

## 原因分析
centos7使用的网卡配置文件是ifcfg-files(`/etc/sysconfig/network-scripts/ifcfg-XXX`)，而RHEL9使用的是keyfiles(`/etc/NetworkManager/system-connections/XXX.nmconnection`)。
升级到Rocky 9后，需要把eth0网卡重新配置一下。
## 解决方法
使用nmcli，先删除'Wired connection 1'这个连接，再为eth0网卡生成新的连接，方法如下：
```
# 删除 Wired connection 1 连接
nmcli dev disconnect eth0
nmcli con del c04f10b8-8831-3d68-9b79-2ac22907357c # nmcli con show 查找`Wired connection 1`连接的uuid，根据uuid删除连接
ip -4 route flush dev eth0

# 重新生成eth0连接 (这里设置static IP 10.206.216.91，网关10.206.216.254, dns 8.8.8.8)
nmcli con add con-name eth0 autoconnect yes  type ethernet ifname eth0 ipv4.method manual ip4 "10.206.216.91/24" gw4 "10.206.216.254"  ipv4.dns "8.8.8.8"
nmcli c up eth0
```

## 参考
[https://www.redhat.com/en/blog/rhel-9-networking-say-goodbye-ifcfg-files-and-hello-keyfiles](https://www.redhat.com/en/blog/rhel-9-networking-say-goodbye-ifcfg-files-and-hello-keyfiles)
