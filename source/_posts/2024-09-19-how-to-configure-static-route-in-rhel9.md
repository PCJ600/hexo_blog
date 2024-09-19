---
layout: next
title: Rocky Linux 9中添加或删除某个网卡的静态路由
date: 2024-09-19 21:37:29
categories: Linux
tags: 
- Linux
- Network
---

## 使用ip命令配置临时路由
添加静态路由
```
ip route add <目的网络> via <下一跳IP> dev <网卡接口名称>
```
例: 给eth0网卡添加一个到达 192.168.2.0/24 网络，下一跳为 192.168.1.254 的路由
```
ip route add 192.168.2.0/24 via 192.168.1.254 dev eth0
```
删除静态路由
```
ip route del <目的网络> via <下一跳IP> dev <网卡接口名称>
```
例: 删除上述的静态路由
```
ip route del 192.168.2.0/24 via 192.168.1.254 dev eth0
```

## 使用nmcli配置永久路由
添加静态路由
例: 给eth0网卡添加一个到达 192.168.2.0/24 网络，下一跳为 192.168.1.254 的路由
```
nmcli connection modify eth0 +ipv4.routes "192.168.2.0/24 192.168.1.254"
```
如果需要添加多个路由，可以用逗号分隔的方式添加：
```
nmcli connection modify eth0 +ipv4.routes "192.168.2.0/24 192.168.1.254,192.168.3.0/24 192.168.1.254"
```
删除静态路由
```
nmcli connection modify eth0 -ipv4.routes "192.168.2.0/24 192.168.1.254"
```

<!-- more -->

## 参考
[Red Hat Documentation —— Chapter 24. Configuring a static route](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/configuring_and_managing_networking/configuring-static-routes_configuring-and-managing-networking#proc_configuring-a-static-route-by-using-nmtui_configuring-static-routes)
