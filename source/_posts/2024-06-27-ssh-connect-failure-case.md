---
layout: next
title: 2024-06-27-ssh-connect-failure-case
date: 2024-08-19 21:51:22
categoires:
- troubleshooting
tags:
- troubleshooting
- Linux
---

## 测试环境
VMware虚拟机 RockyLinux 9 x86_64

双网卡：eth0(访问外网): 10.206.216.92/24; eth1(访问内网) 192.168.1.4/24

## 问题描述
虚拟机重启后，SSH连接失败，提示"Connection time out"，重启之前SSH连接还是正常的。

<!-- more -->

![](image1.png)
## 定位过程
登录Vmware串口，先确认sshd进程是否启动，通过ps查看sshd进程运行进程正常, netstat查看22号端口LISTEN，这一步没问题。
```bash
ps axf | grep sshd
    996 ?        Ss     0:00 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
ss -antp | grep 22 | grep LISTEN
LISTEN    0      128                   0.0.0.0:22                    0.0.0.0:*     users:(("sshd",pid=996,fd=3))
```

再通过`tcpdump -i eth0 port 22` 抓包，确认是否收到SSH请求报文，排除防火墙设置的原因。
![](image2.png)

从抓包结果看，只有收的包，没有发出去的包。再用`route`命令查看路由表，发现问题：**eth0默认路由Metric值为101，大于eth1的Metric(100)，这说明eth1默认路由优先级更高，结果应答报文从eth1发出去了，导致客户端收不到应答，显示"Connection time out"**
![](image3.png)

通过`dmesg`查看虚拟机启动日志，发现eth1居然先于eth0 UP，所以系统自动分配给eth1的metric更小(优先级更高)。而之前SSH连接正常的时候，都是eth0先UP。
![](image4.png)
## 解决方法
手动设置下两个网卡的metric值，保证eth0的`ipv4.route-metric`小于eth1就行。写个脚本每次启动跑一下：
```bash
nmcli connection modify eth0 ipv4.route-metric 100
nmcli connection modify eth1 ipv4.route-metric 101
nmcli c up eth0
nmcli c up eth1
nmcli c reload
```
查看网卡配置文件确认修改生效：
![image5.png]()

