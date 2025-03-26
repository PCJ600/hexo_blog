---
layout: next
title: Microk8s启动失败，Failed to start Service for snap application microk8s.daemon-containerd 问题定位
date: 2024-05-21 20:43:20
categories: Kubernetes
tags:
- troubleshooting
- Kubernetes
---

## 测试环境
VMware Rocky Linux 9.3 x86_64

## 问题描述
microk8s启动失败，microk8s.daemon-containerd服务启动失败，报“Failed to start Service for snap application microk8s.daemon-containerd”

## 定位过程
<!-- more -->
1. 敲`microk8s inspect`，检查相关service启动是否正常，发现snap.microk8s.daemon-containerd服务没有running.
![](image1.png)
2. 敲`journal -u snap.microk8s.daemon-containerd`，查看服务启动失败原因：
![](image2.png)
3. 观察上面的journal日志发现，microk8s.daemon-containerd反复尝试获取默认路由失败，因此怀疑是网卡配置有问题，敲`route`发现路由表为空，敲`ifconfig`发现主网卡没有配置IP地址。
![](image3.png)
4. 检查VMware虚拟机设置，找到网卡没有IP地址原因：主网卡没有**connect**
![](image4.png)
## 解决方法
Vmware VM设置里勾选网卡状态为connect，再设置主网卡IP, Gateway, DNS，重启microk8s问题解决。
配置网卡使用`nmcli`命令，方法如下：
* 修改已存在的connection
```bash
nmcli con mod eth0 ipv4.addresses "${ip}"
nmcli con mod eth0 ipv4.dns "${dns}"
nmcli con mod eth0 ipv4.gateway "${gw}"
nmcli con up eth0
nmcli c reload
```
* connection不存在，需要新增
```bash
nmcli con add con-name eth0 autoconnect yes  type ethernet ifname eth0 ipv4.method manual ip4 "${ip}" gw4 "${gw}" ipv4.dns "${dns}"
nmcli c up eth0
nmcli c reload
```
