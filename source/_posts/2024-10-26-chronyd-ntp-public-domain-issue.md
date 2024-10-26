---
layout: next
title: chronyd配置了local的NTP server后, NTP报文出现public ip的问题
date: 2024-10-26 13:12:59
categories: troubleshooting
tags: 
- Linux
- chronyd
---

## 问题描述
客户在Rocky Linux 9.4的VM上配了一个local的NTP server(IP: 10.64.1.76)。配置完成后, 时钟可以同步，但一段时间后客户的firewall收到告警, 拒绝了大量目标端口为123的请求, 且这些请求的目的IP并不是客户指定的NTP server的IP，客户要求解释原因并给出解决方案。

## 定位
先通过`chronyc sourcestats`命令，查看当前连接的NTP服务器

<!-- more -->

```
Name/IP Address            NP  NR  Span  Frequency  Freq Skew  Offset  Std Dev
==============================================================================
10.64.1.76                  1   0     0     +0.000   2000.000    +15us  4000ms
117.80.112.205              0   0     0     +0.000   2000.000     +0ns  4000ms
202.112.29.82               0   0     0     +0.000   2000.000     +0ns  4000ms
tock.ntp.infomaniak.ch      0   0     0     +0.000   2000.000     +0ns  4000ms
2001:da8:9000::130          2   0     2     +0.000   2000.000  -1254us  4000ms
ntp1.flashdance.cx          0   0     0     +0.000   2000.000     +0ns  4000ms
tock.ntp.infomaniak.ch      0   0     0     +0.000   2000.000     +0ns  4000ms
2001:470:1d:281::123        1   0     0     +0.000   2000.000    -12ms  4000ms
time.cloudflare.com         1   0     0     +0.000   2000.000  -2445us  4000ms
```
结果显示, 除了客户指定的NTP server(10.64.1.76), chronyd的确连接了其他的server, 通过tcpdump抓包也能看出NTP报文中出现了public IP
```
[root@localhost ~]# tcpdump -i eth0 dst port 123 -n
07:14:03.968036 IP6 2620:101:4002:800b:456f:575f:396d:fe0d.30612 > 2001:da8:9000::130.ntp: NTPv4, Client, length 48
07:14:05.261407 IP6 2620:101:4002:800b:456f:575f:396d:fe0d.34042 > 2606:4700:f1::123.ntp: NTPv4, Client, length 48
07:14:05.850132 IP 10.206.216.92.63453 > 10.64.1.76.ntp: NTPv4, Client, length 48
07:14:06.383730 IP6 2620:101:4002:800b:456f:575f:396d:fe0d.39439 > 2001:470:1d:281::123.ntp: NTPv4, Client, length 48
07:15:08.229127 IP6 2620:101:4002:800b:456f:575f:396d:fe0d.57634 > 2001:da8:9000::130.ntp: NTPv4, Client, length 48
07:15:10.499490 IP6 2620:101:4002:800b:456f:575f:396d:fe0d.52853 > 2606:4700:f1::123.ntp: NTPv4, Client, length 48
```

检查配置文件/etc/chrony.conf, 发现同时配置了server和pool, 如下:
```
server 10.64.1.76 iburst
pool 2.rocky.pool.ntp.org iburst
```
查阅资料发现，当同时配了server和pool时，chrony会同时向server以及NTP pool中的server发送请求，根据一定的算法和策略选择最准确可靠的源来同步系统时钟。 默认的`2.rocky.pool.ntp.org`是一个public的NTP pool，所以客户防火墙deny了这些public IP.

想搞清楚NTP选择source的策略，需要下一份源码调试，后面有时间再来研究
```
# chronyd --version  确认chronyd版本
chronyd (chrony) version 4.3
源码: https://git.tuxfamily.org/chrony/chrony.git/
```

## 解决方法
* 让客户放开这些public IP，或者123端口(这种做法显然不安全)
* 或者禁用pool, 修改/etc/chrony.conf, 删掉`pool 2.rocky.pool.ntp.org iburst`, 再`systemctl restart chronyd`重启chronyd. (这种方法会降低时间同步的可靠性，如果local server不工作，时间同步就会失败)

## 参考
[https://chrony-project.org/index.html](https://chrony-project.org/index.html)
