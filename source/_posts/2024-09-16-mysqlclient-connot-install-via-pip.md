---
layout: next
title: Rocky Linux 9安装mysqlclient库报错的解决方法
date: 2024-09-16 17:03:17
categories: Django
tags: Django
---

## 环境
VMware Rocky Linux 9.4 MySQL 8.0
## 安装mysqlclient报错
```
yum install python3-devel
pip3 install mysqlclient
```
报错：
```
Downloading http://mirrors.aliyun.com/pypi/packages/37/fb/d9a8f763c84f1e789c027af0ffc7dbf94c9a38db961484f253f0552cbb47/mysqlclient-2.2.1.tar.gz (89 kB)
     |████████████████████████████████| 89 kB 80.1 MB/s
  Installing build dependencies ... done
  Getting requirements to build wheel ... error
  ERROR: Command errored out with exit status 1:
   command: /usr/bin/python3 /usr/lib/python3.9/site-packages/pip/_vendor/pep517/in_process/_in_process.py get_requires_for_build_wheel /tmp/tmp5fvp1dau
       cwd: /tmp/pip-install-1nnewfot/mysqlclient_93347d191d2942c8b2bb37681a22fd09
  Exception: Can not find valid pkg-config name.
  Specify MYSQLCLIENT_CFLAGS and MYSQLCLIENT_LDFLAGS env vars manually
```

<!-- more -->

## 解决方法
需要先安装mysql-devel这个RPM包，再用pip安装mysqlclient，操作如下：
在 [https://dev.mysql.com/downloads/repo/yum/](https://dev.mysql.com/downloads/repo/yum/) 找到MySQL 8.0版本对应的`mysql84-community-release-el9-1.noarch.rpm`，把这个RPM包拷到VM里手动安装
```
rpm -i mysql84-community-release-el9-1.noarch.rpm
yum install -y python3-devel mysql-devel
pip3 install mysqlclient
```
`pip list` 命令查看已安装的mysqlclient信息
```
pip list  | grep mysqlclient
mysqlclient         2.2.4
```

## 参考
[https://stackoverflow.com/questions/76585758/mysqlclient-cannot-install-via-pip-cannot-find-pkg-config-name-in-ubuntu](https://stackoverflow.com/questions/76585758/mysqlclient-cannot-install-via-pip-cannot-find-pkg-config-name-in-ubuntu)
