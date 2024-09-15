---
layout: next
title: Django基础
date: 2024-09-15 15:17:53
categories: Django
tags: Django
---

## Django快速上手
参考: [Django快速上手](https://pcj600.github.io/2024/08/23/2024-08-23-how-to-install-django-on-rhel9/)

## 再写几个页面
编辑`demo1/urls.py`, 添加URL和视图函数映射
```py
urlpatterns = [
    path('index/', views.index),
	path('user/list/', views.user_list),
	path('user/add/', views.user_add),
]
```

编辑`app01/views.py`，添加几个函数
```py
from django.shortcuts import render, HttpResponse

# Create your views here.
def index(request):
    return HttpResponse("Hello World")

def user_list(request):
    return HttpResponse("User List")

def user_add(request):
    return HttpResponse("User add")
```

## templates模板的运用
<!-- more -->
编辑`app01/views.py`，使用render返回一个HTML页面
```py
def user_list(request):
	return render(request, "user_list.html")
```

app01目录下创建`templates/user_list.html`
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    <h1>User List</h1>
</body>
</html>
```

## 引用静态文件
在app目录下创建static目录，image、css、js都放在static目录下，static目录结构:
```
static
|- css
|- img
|- js
|- plugins
```

引用Bootstrap, JQuery, image, 编辑`templates/user_list.html`
```html
{% load static %}

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
    <link rel="stylesheet" href="{% static 'plugins/bootstrap-3.4.1/css/bootstrap.css' %}">
</head>
<body>
    <h1>User List</h1>

    <input type="text" class="btn btn-primary" value="Create" />

    <img src="{% static 'img/1.png' %}" alt="" />

    <script src="{% static 'js/jquery-3.7.1.min.js' %}"></script>
    <script src="{% static 'plugins/bootstrap-3.4.1/js/bootstrap.js' %}"></script>
</body>
</html>
```

Q: 为什么使用load static这种方式引入静态文件？/static/img/1.png不也行吗？
A: 如果静态文件移动到别的路径，只需要改settings.py的配置，不需要逐个修改每个页面的路径

