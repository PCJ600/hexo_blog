---
layout: next
title: Linux双网卡环境概率出现DNS解析错误
date: 2024-03-27 20:10:19
categories:
- troubleshooting
tags:
- troubleshooting
- Network
---

## 测试环境
VMware Rocky Linux 9 虚拟机, 双网卡(eth0和eth1)配置如下：
```
eth0 10.206.216.27/24  DNS 10.204.16.18
eth1 192.168.1.27/24   DNS 192.168.1.1
```
## 问题描述
手动配置eth1的DNS后，网络不通，通过抓包发现是eth1的DNS server配置有误，**但机器重启后网络有概率能恢复正常**。
<!-- more -->
## 定位方法
在网络不通的时候，查看`/etc/resolv.conf`内容：
![](image1.png)
找到原因：**libc解析器不支持超过3条以上的nameserver**，网络不通的时候，可以正常工作的nameserver 10.204.16.18恰好在第4条，所以DNS解析失败。

解决方法：删掉第二、三行这两个系统自动生成的v6 dns server，保证`resolv.conf`里的nameserver记录不超过3条即可。

注意不要直接修改`/etc/resolv.conf`，RHEL9上删除nameserver可以使用`nmcli`，如下：
```
nmcli con mod eth0 ipv6.ignore-auto-dns yes
nmcli con mod eth1 ipv6.ignore-auto-dns yes
systemctl restart NetworkManager
```
## 参考资料
[man resolv.conf](https://man7.org/linux/man-pages/man5/resolv.conf.5.html)
