---
layout: next
title: 在RHEL9上安装MySQL
date: 2024-09-16 14:47:56
categories: MySQL
tags: MySQL
---

## 安装并启动MySQL
```bash
dnf install -y mysql-server
systemctl start mysqld.service
```

## 设置MySQL的root用户密码
```bash
mysql_secure_installation
```
`mysql_secure_installation`是MySQL的一个安全脚本，执行后根据提示选择密码强度，输入root用户的密码
## 查看MySQL的版本
通过mysqladmin查到MySQL版本为8.0.36
```bash
mysqladmin -u root -p version
Enter password:

....
Server version          8.0.36
```
## 连接MySQL
```bash
mysql -u root -p
```
再输入你之前设置的root用户密码即可。

<!-- more -->
## 参考
[https://juejin.cn/post/7162834297739542565](https://juejin.cn/post/7162834297739542565)
