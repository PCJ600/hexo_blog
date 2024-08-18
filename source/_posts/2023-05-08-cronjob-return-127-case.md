---
layout: next
title: crontab执行失败返回127的原因和解决方法
date: 2023-05-08 21:26:43
categories:
- Linux
tags: 
- Linux
- troubleshooting
- crontab
---

## 问题描述
用`crontab`执行自己编写的Shell脚本报错，`iptables`命令返回127(command not found) ，在后台手动执行脚本是正常的。
## 解决方法
添加PATH，在脚本里添加如下：
```bash
export PATH=/usr/sbin/:$PATH
```
<!-- more -->
## 参考资料
[https://blog.csdn.net/qq_36588424/article/details/127549734](https://blog.csdn.net/qq_36588424/article/details/127549734)
