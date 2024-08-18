---
layout: next
title: 使用systemd启动stunnel，并设置自动重启的方法
date: 2023-04-26 21:25:21
categories: systemd
tags: systemd
---

## 操作步骤

运行环境: Centos7

1. 安装stunnel 
```
yum install stunnel
```

2. 编写systemd配置文件，放到`/usr/lib/systemd/system`

<!-- more -->
```txt
[Unit]
Description=Stunnel Proxy Server
After=network.target

[Service]
Type=forking
User=root
ExecStart=/usr/bin/stunnel /etc/stunnel/stunnel.conf
Restart=always

[Install]
WantedBy=multi-user.target
```

参数解释：
* Restart=always 只要不是通过`systemctl stop`来停止服务，任何情况下都必须要重启服务，默认值为no

3. 启动stunnel
```bash
systemctl daemon-reload
systemctl start stunnel
```

## 手动启动和停止stunnel的方法
启动stunnel
```bash
stunnel /etc/stunnel/stunnel.conf
```
停止stunnel
```bash
kill `cat /var/run/stunnel.pid`
```

## 参考
[https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/security_guide/sect-starting_stopping_restarting_stunnel](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/security_guide/sect-starting_stopping_restarting_stunnel)
