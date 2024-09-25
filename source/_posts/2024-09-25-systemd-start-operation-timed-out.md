---
layout: next
title: 'Systemd服务启动报错: Start operation timed out(执行systemctl start后卡住)的解决方法'
date: 2024-09-25 19:41:48
categories: systemd
tags: systemd
---

## 问题描述
在RHEL中运行了一个自定义的systemd服务，启动报错` Start operation timed out`, 在后台执行systemctl start也被阻塞, 不能自动退出。

service配置文件如下:
```
[Unit]
After=network.target

[Service]
Type=forking
ExecStart=/path/to/monitor_network.sh

[Install]
WantedBy=multi-user.target
```

## 解决方法
查阅资料，发现Type=forking有问题，这里应该改成Type=simple。

<!-- more -->

* Type=forking时，systemd会认为/path/to/monitor_network.sh是一个守护进程，systemd会等到这个守护进程(父进程)调用fork生成子进程，父进程退出后，systemd才退出。但是我这里的进程是一个不会退出的前台进程，并不是守护进程，所以systemd一直处于等待状态，报`Start operation timed out`。 
* 这个问题还有一个错误现象： 你在后台执行systemctl start会被阻塞，尽管后台可以查到进程运行; 通过systemctl status可以查到服务的状态是start，而不是running。

关于Systemd Type的详细介绍请移步：[Systemd中文手册](https://www.jinbuguo.com/systemd/systemd.service.html)

## 参考
[https://stackoverflow.com/questions/45012415/systemd-start-operation-timed-out-terminating](https://stackoverflow.com/questions/45012415/systemd-start-operation-timed-out-terminating)
[https://www.cnblogs.com/niway/p/15346572.html](https://www.cnblogs.com/niway/p/15346572.html)
