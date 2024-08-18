---
layout: next
title: Traceroute简介和抓包
date: 2023-05-15 21:43:51
categories: Network
tags:
- Linux
- Traceroute
- Network
---

## traceroute原理
Traceroute 是一种网络诊断工具，用于确定数据包从一个源到达目的地所经过的路径。它通过发送一系列的数据包，并观察每个数据包经过的路由器（或称为跃点）来实现这一目的。

<!-- more -->

**工作原理：**

* Traceroute 发送一系列的 UDP 数据包，每个数据包都具有不同的 TTL（Time to Live）值。TTL 值表示数据包在网络中可经过的最大跃点数。
* 当第一个数据包离开源设备时，它的 TTL 值被设置为 1。当该数据包到达网络中的第一个路由器时，路由器会将 TTL 值减 1，并转发该数据包。如果 TTL 值减为 0，路由器将丢弃该数据包并向源设备发送 ICMP 时间超时报文。
* 源设备收到 ICMP 时间超时报文后，就知道了到达第一个路由器的路径。然后，源设备增加 TTL 值并发送第二个数据包，以便确定下一个路由器。
* 这个过程不断重复，每次 TTL 值增加，直到数据包到达目的地。目的设备将收到数据包并向源设备发送 ICMP 端口不可达报文，从而终止 Traceroute 进程。
* Traceroute 将记录每个数据包的路径，包括每个跃点的 IP 地址和响应时间。最终将呈现给用户一张路由跟踪表，显示了数据包从源到目的地经过的所有路由器。

总的来说，Traceroute 通过探测数据包的路径和响应时间，帮助网络管理员诊断网络连接问题，定位潜在的瓶颈或故障点。
## traceroute用法
Linux系统下，traceroute用法：
```bash
traceroute hostname
```
Windows系统使用tracert命令:
```bash
tracert hostname
```
## traceroute抓包
在宿主机10.206.216.95上执行`traceroute www.baidu.com`，通过`tcpdump`抓包，如下：
```bash
$ ifconfig ens192
ens192: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.206.216.95  netmask 255.255.255.0  broadcast 10.206.216.255
route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         gateway         0.0.0.0         UG    100    0        0 ens192
10.206.216.0    0.0.0.0         255.255.255.0   U     100    0        0 ens192

# tmux pane 1
$ tcpdump -i ens192 icmp and host 10.206.216.95 -w traceroute.pcap

# tmux pane 2
$ traceroute  www.baidu.com
traceroute to www.baidu.com (36.155.132.55), 30 hops max, 60 byte packets
 1  gateway (10.206.216.254)  0.239 ms  0.152 ms  0.147 ms
 2  10.206.2.254 (10.206.2.254)  0.265 ms  0.274 ms  0.241 ms
 3  192.168.200.1 (192.168.200.1)  0.545 ms  0.506 ms  0.825 ms
 4  36.152.113.193 (36.152.113.193)  2.837 ms  3.182 ms  3.552 ms
 5  221.178.162.185 (221.178.162.185)  2.632 ms * *
 6  * * 183.207.54.161 (183.207.54.161)  3.116 ms
 7  183.207.67.106 (183.207.67.106)  3.576 ms 183.207.22.130 (183.207.22.130)  3.195 ms 183.207.66.102 (183.207.66.102)  3.638 ms
 8  * 36.155.156.46 (36.155.156.46)  3.884 ms 36.155.156.54 (36.155.156.54)  4.690 ms
 9  36.155.157.174 (36.155.157.174)  4.197 ms 36.155.157.170 (36.155.157.170)  4.344 ms  3.964 ms
10  * * *
11  * * *
12  * * *
13  * * *
14  * * *
15  * * *
16  * * *
17  * * *
18  * * *
19  * * *
20  * * *
21  * * *
22  * * *
23  * * *
24  * * *
25  * * *
26  * * *
27  * * *
28  * * *
29  * * *
30  * * *
```
wireshark分析结果：
![](image1.png)




