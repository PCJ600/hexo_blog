---
layout: next
title: 如何设置systemd服务自动重启
date: 2023-05-09 21:35:06
categories: systemd
tags: systemd
---

## 设置systemd服务自动重启的方法
service文件里添加如下配置：
```
Restart=always
RestartSec=30
StartLimitInterval=0
```
* Restart=always: 只要不是通过systemctl stop来停止服务，任何情况下都必须要重启服务，默认值为no
* RestartSec=30: 重启间隔，比如某次异常后，等待30(s)再进行启动，默认值0.1(s)
* StartLimitInterval=0: 无限次重启，默认是10秒内如果重启超过5次则不再重启，设置为0表示不限次数重启

<!-- more -->
## 参考
【1】[https://www.freedesktop.org/software/systemd/man/systemd.service.html](https://www.freedesktop.org/software/systemd/man/systemd.service.html)
【2】[https://blog.csdn.net/easylife206/article/details/101730416](https://blog.csdn.net/easylife206/article/details/101730416)
