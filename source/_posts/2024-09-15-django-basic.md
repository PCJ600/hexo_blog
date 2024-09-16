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


## Django模板语法
什么是Django模板: 在HTML中写一些占位符，由数据对占位符进行替换和处理

举例:
编辑`app01/views.py`
```py
def tpl(request):
    name = 'Peter'
    roles = ['admin', 'guest']
    user_info = {'name': 'Tony', 'salary': 10000, 'role': 'CEO'}
	data_list = [
        {"name": "peter", "salary": 10000, "role": "CTO"},
        {"name": "tony", "salary": 5000, "role": "CFO"}
    ]
    return render(request, 'tpl.html', {"n1": name, "n2": roles, "n3": user_info, "n4": data_list})
```
编辑`demo1/urls.py`
```py
urlpatterns = [
    path('user/tpl/', views.tpl),
]
```
新增`app01/templates/tpl.html`
```html
<body>
    <div>{{ n1 }}</div>
    <div>{{ n2 }}</div>
    <div>{{ n2.0 }}</div>
    <div>{{ n2.1 }}</div>
	<div>
        {% for item in n2 %}
            <span>{{ item }}</span>
        {% endfor %}
    </div>
	<hr/>
	
    {{ n3.name }}
    {{ n3.salary }}
    {{ n3.role }}
    {% for k,v in n3.items %}
        <li>{{ k }} == {{ v }} </li>
    {% endfor %}
    {% for k in n3.keys %}
        <li>{{ k }} </li>
    {% endfor %}
    {% for v in n3.values %}
        <li>{{ v }} </li>
    {% endfor %}
	
	<div>{{ n4 }}</div>
    {% for item in n4 %}
        <div>{{ item.name }}, {{ item.salary }}</div>
    {% endfor %}
	
	{% if n1 == 'Peter' %}
        <div>Peter!</div>
    {% else %}
        <div>Not Peter!</div>
    {% endif %}
</body>
```

curl localhost:8000/user/tpl/
```
Peter
['admin', 'guest']
admin
guest
admin guest
Tony 10000 CEO
name == Tony
salary == 10000
role == CEO
name
salary
role
Tony
10000
CEO
[{'name': 'peter', 'salary': 10000, 'role': 'CTO'}, {'name': 'tony', 'salary': 5000, 'role': 'CFO'}]
peter, 10000
tony, 5000
Peter!
```

## 案例：简单的用户登录(无数据库)
编辑demo1/urls.py, 添加映射
```py
from app01 import views
urlpatterns = [
    path('login/', views.login),
]
```
编辑app01/views.py, 实现login函数
```py
def login(request):
    if request.method == "GET":
        return render(request, "login.html")

    username = request.POST.get("user")
    password = request.POST.get("pwd")
    if username == "root" and password == "123":
        return redirect("https://www.baidu.com")
    return render(request, "login.html", {"error_msg": "Login Failed"})
```
新增app01/template/login.html
```html
{% load static %}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    <h1>User Login</h1>

    <form method="post" action="/login/">
        <input type="text", name="user", placeholder="Username"/>
        <input type="password", name="password", placeholder="Password"/>
        <input type="submit" value="Submit"/>
    </form>
</body>
</html>
```
提交表单后报错: CSRF verification failed, Forbidden(403)
解决方法: 在form表单里加`{% csrf_token %}`
```html
	<h1>User Login</h1>
    <form method="post" action="/login/">
		{% csrf_token %}
        <input type="text", name="user", placeholder="Username"/>
        <input type="password", name="password", placeholder="Password"/>
        <input type="submit" value="Submit"/>
    </form>
```




