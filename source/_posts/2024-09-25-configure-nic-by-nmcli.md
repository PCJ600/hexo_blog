---
layout: next
title: 使用nmcli配置某个网卡的static IP,Gateway,DNS的方法
date: 2024-09-25 19:39:01
categories: Linux
tags: Linux
---

先查看当前网卡
```
nmcli con show
NAME	UUID	TYPE 		DEVICE
eth0	XX	ethernet 	eth0
lo	XX	loopback	lo
```

例如，配置eth0网卡的static ip为10.206.216.93, gateway 10.206.216.254, DNS 10.204.16.18
```
nmcli con del eth0
nmcli con add con-name eth0 autoconnect yes type ethernet ifname eth0 ip4 10.206.216.93/24 gw4 10.206.216.254 ipv4.dns "8.8.8.8 4.4.4.4"
nmcli con up eth0
```
