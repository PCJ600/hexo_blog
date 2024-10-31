---
layout: next
title: MySQL8.0如何重置密码
date: 2024-10-17 20:24:36
categories: MySQL
tags: MySQL
---

## 测试环境
* VMware Rocky Linux 9.4
* MySQL 8.0.36

## 修改MySQL配置文件, 设置为免密登录
修改`/etc/my.cnf.d/mysql-server.cnf`, 在[mysqld]结尾添加一行`skip-grant-tables`
<!-- more -->
```
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
log-error=/var/log/mysql/mysqld.log
pid-file=/run/mysqld/mysqld.pid
skip-grant-tables
```

## 重启MySQL服务
```
systemctl restart mysqld
```

## 连接MySQL，修改密码
```
mysql
```
连接成功后，执行如下sql命令
```
use mysql;
flush privileges;
alter user 'root'@'localhost' identified by 'your_new_password';
grant all privileges on *.* to "root"@'localhost';
flush privileges;
```

## 还原配置文件并重启MySQL服务
修改`/etc/my.cnf.d/mysql-server.cnf`, 删掉开头加的`skip-grant-tables`，再重启服务即可
```
systemctl restart mysqld
```

## 参考
[https://blog.csdn.net/Box_clf/article/details/124599166](https://blog.csdn.net/Box_clf/article/details/124599166)
[https://blog.namichong.com/learn/sql/mysql/mysql_8.0_reset_password.html](https://blog.namichong.com/learn/sql/mysql/mysql_8.0_reset_password.html)

