---
layout: next
title: RHEL9中找不到/var/log/dmesg日志文件问题
date: 2024-07-03 22:00:18
categories: Linux
tags: Linux
---

## 问题描述
在Rocky Linux 9 服务器上查看启动日志，发现没有`/var/log/dmesg`文件。

## dmesg是什么？
`dmesg`(diagnostic messages)用于打印`kernel ring buffer`的所有消息。 kernel会将开机信息存储在ring buffer中，如果开机时来不及查看启动信息，可以通过`dmesg`命令查看。

`Kernel ring buffer`（内核环形缓冲区）是一种在Linux内核中使用的数据结构，用于在生产者（如硬件设备、驱动程序或内核线程）和消费者（如用户空间应用程序）之间传输数据。 这种缓冲区通常用于日志记录、性能监控、事件跟踪等场景。

<!-- more -->

## RHEL9找不到/var/log/dmesg日志文件的原因
参考：[https://access.redhat.com/solutions/3748981](https://access.redhat.com/solutions/3748981)，这里摘录如下：

By design, the /var/log/dmesg file is not generated during boot. The kernel ring buffer is captured within the systemd-journal as well as /var/log/messages, via the imjournal rsyslog plugin.

翻译一下： 根据设计，在RHEL8/RHEL9中，启动期间不会自动生成`/var/log/dmesg`文件。 内核环形缓冲区(kernel ring buffer)的消息通常会被systemd-journa捕获，存储在`/var/log/messages`或`journalctl`中。这种设计可以提供更现代化的日志管理方式，支持更好的搜索和过滤功能
## 解决方法
手动执行`dmesg`生成日志文件
```
dmesg > /var/log/dmesg
```
或者使用 `journalctl`命令
```
journalctl -k | tee /var/log/dmesg
```

## 参考
【1】[https://en.wikipedia.org/wiki/Dmesg](https://en.wikipedia.org/wiki/Dmesg)
【2】[https://www.getpagespeed.com/solutions/the-var-log-dmesg-file-is-not-created-during-boot-for-rocky-linux-8](https://www.getpagespeed.com/solutions/the-var-log-dmesg-file-is-not-created-during-boot-for-rocky-linux-8)
