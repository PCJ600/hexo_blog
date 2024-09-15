---
layout: next
title: Django快速上手
date: 2024-08-23 23:02:54
Categories: Django
tags:
- Django
- Python
---

## 更新软件包
```
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

## 创建一个Django项目
### 1. 创建项目
`django-admin startproject demo1`
```
tree
├── demo1
│   ├── asgi.py		 无需修改, 接收网络请求
│   ├── __init__.py
│   ├── settings.py     【重要】 项目配置文件
│   ├── urls.py		【重要】URL和function的映射
│   └── wsgi.py		 无需修改, 接收网络请求
└── manage.py		 无需修改, 项目管理，启动项目，创建APP，数据管理
```
### 2. 创建APP
`python3 manage.py startapp app01`
```
tree
├── app01
│   ├── admin.py         不用修改，django默认提供的admin后台管理
│   ├── apps.py          不用修改
│   ├── migrations       不用修改, 数据库变更
│   ├── models.py       【重要】ORM 对数据库操作
│   ├── tests.py         不用修改，这个是单元测试
│   └── views.py	【重要】写函数的
├── demo1
│   ├── asgi.py
│   ├── __init__.py
│   ├── settings.py     【重要】项目配置文件
│   ├── urls.py		【重要】URL->函数
│   └── wsgi.py
└── manage.py
```
### 3. 快速上手，写一个页面
1. 编辑`demo1/settings.py`, INSTALLED_APPS中添加'app01.apps.App01Config'
```py
# Application definition
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
	# ...
    'app01.apps.App01Config', # Add your app config here !
]
```

2. 编辑`demo1/urls.py`, 添加URL和视图函数的映射
```py
from app01 import views
urlpatterns = [
    path('index/', views.index),
]
```

3. 编辑`app01/views.py`
```py
from django.shortcuts import render, HttpResponse

def index(request):
	return HttpResponse("Hello World")
```
### 4. 启动项目，测试
启动：`python3 manage.py runserver 0.0.0.0:8000`
测试：`curl localhost:8000/index/`

## 参考
[https://docs.djangoproject.com/zh-hans/5.1/topics/install/#installing-official-release](https://docs.djangoproject.com/zh-hans/5.1/topics/install/#installing-official-release)



