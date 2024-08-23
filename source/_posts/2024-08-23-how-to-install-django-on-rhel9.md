---
layout: next
title: 如何在Rocky Linux 9上安装Django
date: 2024-08-23 23:02:54
Categories: Django
tags:
- Django
- Python
---

## 更新软件包
```bash
dnf check-update
dnf install dnf-utils
```
## 安装Python3和pip
```
dnf install python3 python3-pip
```
## 通过pip安装Django
```
pip3 install Django
```

## 验证Django安装是否成功
```
django-admin --version
4.2.15
```
<!-- more -->

可以显示版本号(4.2.15)，说明安装成功
## 创建一个简单Django项目，并运行服务器
```
django-admin startproject myproject
cd myproject
python3 manage.py runserver 0.0.0.0:8000
```

通过`curl localhost:8080`测试服务器，返回200 OK

## 参考
[https://docs.djangoproject.com/zh-hans/5.1/topics/install/#installing-official-release](https://docs.djangoproject.com/zh-hans/5.1/topics/install/#installing-official-release)



