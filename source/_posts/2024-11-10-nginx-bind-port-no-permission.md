---
layout: next
title: 'Nginx bind() to 0.0.0.0:88 failed (13: Permission denied)解决方法'
date: 2024-11-10 14:11:08
categories: Nginx
tags: Nginx
---

## 问题描述
我在Nginx上添加一个端口号为88的虚拟主机， 重新启动Nginx报错：` bind() to 0.0.0.0:88 failed (13: Permission denied)`

## 解决方法
查阅资料，发现这类bind无权限问题，大多由SElinux引起。SELinux有三种模式，如下：
* enforcing：强制模式，此时SELinux 运行中
* permissive：宽容模式，此时SELinux运行中，会像 enforcing 模式一样加载安全策略，但不会拒绝任何操作。
* disabled：关闭，SELinux 没有运行。

<!-- more -->

查看当前SELinux状态，为enforcing
```
getenforce
Enforcing
```
这里需要使用`setenforce`把SELinux模式改成Permissive
```
setenforce 0
getenforce
Permissive
```
再重启Nginx, 问题解决
```
systemctl restart nginx
```
## 参考
【1】[https://docs.redhat.com/zh_hans/documentation/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/sect-security-enhanced_linux-introduction-selinux_modes](https://docs.redhat.com/zh_hans/documentation/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/sect-security-enhanced_linux-introduction-selinux_modes)
【2】[https://blog.csdn.net/qq_45663927/article/details/134857385](https://blog.csdn.net/qq_45663927/article/details/134857385)
