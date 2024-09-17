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

## Django连接MySQL数据库
框架：业务代码 -> ORM -> (pymysql,MySQLdb,mysqlclient) -> Database

### 安装MySQL
参考: [https://pcj600.github.io/2024/09/16/2024-09-16-how-to-install-mysql-on-rhel9/](https://pcj600.github.io/2024/09/16/2024-09-16-how-to-install-mysql-on-rhel9/)

### 安装mysqlclient
参考: [https://pcj600.github.io/2024/09/16/2024-09-16-mysqlclient-connot-install-via-pip/](https://pcj600.github.io/2024/09/16/2024-09-16-mysqlclient-connot-install-via-pip/)

### ORM
* 支持创建、修改、删除表(不用你写SQL语句), 但无法创建数据库
* 操作表中的数据(不用你写SQL语句)

### 创建数据库
```
mysql -u root -p 
create database gx_day15 DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
```
* DEFAULT CHARSET utf8: 指定了数据库的默认字符集。
* COLLATE utf8_general_ci: 表示使用 utf8 字符集的不区分大小写的校对规则(ci 表示 case-insensitive)s

### 连接数据库
[https://docs.djangoproject.com/en/5.1/ref/databases/#mysql-notes](https://docs.djangoproject.com/en/5.1/ref/databases/#mysql-notes)

编辑demo1/settings.py
```py
DATABASES = {
	'default' = {
		'ENGINE': 'django.db.backends.mysql',
		'NAME': 'gx_day15',
		'USER': 'root',
		'PASSWORD': 'XXX',
		'HOST': 'localhost',
		'PORT': 3306,
	}
}
```
### 创建、修改表

创建表不需要写SQL语句，只需在app01/models.py里定义一个类
```py
class UserInfo(models.Model):
	name = models.CharField(max_length=32)
	password = models.CharField(max_length=64)
	age = models.IntergerField(default=2)
```
相当于创建了一个表，表名: app01_userinfo
```sql
create table app01_userinfo(
	id bigint auto_increment primary key,
	name varchar[32],
	password varchar[64],
	age int
)
```
执行命令，让Django真正创建表。
先确认APP已经注册：demo1/settings.py
```py
INSTALLED_APPS = [
    'app01.apps.App01Config',
]
```
在项目根目录执行:
```bash
python3 manage.py makemigrations
python3 manage.py migrate
```

查看表
```
mysql -u root -p 
mysql> use gx_day15;
mysql> desc app01_userinfo;
+----------+-------------+------+-----+---------+----------------+
| Field    | Type        | Null | Key | Default | Extra          |
+----------+-------------+------+-----+---------+----------------+
| id       | bigint      | NO   | PRI | NULL    | auto_increment |
| name     | varchar(32) | NO   |     | NULL    |                |
| password | varchar(64) | NO   |     | NULL    |                |
| age      | int         | NO   |     | NULL    |                |
+----------+-------------+------+-----+---------+----------------+
4 rows in set (0.01 sec)
```

表中新增列时，由于已存在列中可能已有数据，所以新增列必须要指定新增列对应的数据:
* 手动输入一个值
* 设置默认值

删除表: 在models.py里删掉对应的类，再执行
```py
python3 manage.py makemigrations
python3 manage.py migrate
```

### 创建表中的数据
app01/models.py
```py
class Department(models.Model):
    title = models.CharField(max_length=16)
```
app01/views.py
```py
from app01.models import Department,UserInfo
def orm(request):
    Department.objects.create(title='sales')
    return HttpResponse('ORM OK')
```

### 删除表中数据
app01/views.py
```py
from app01.models import Department,UserInfo
def orm(request):
    Department.objects.filter(title='sales').delete() # 删掉所有title=sales的数据
	Department.objects.all().delete() # 所有数据都删掉
    return HttpResponse('ORM OK')
```

### 更新表中数据
```py
data_list = UserInfo.objects.all()
for obj in data_list:
	print(obj.id, obj.name, obj.password)
	
	# data_list = UserInfo.objects.filter(id=1)
	o = UserInfo.objects.filter(id=1).first()
	print(o.id, o.name, o.password)
	
	# 更新所有行的数据
	UserInfo.objects.all().update(password='123456')

	UserInfo.objects.filter(name='peter').update(password='123456')
```
* objects.all()返回queryset类型，每个元素是一个对象
* UserInfo.objects.filter(id=1).first() 返回符合筛选条件的第一条数据


## 数据库操作的案例 —— 用户管理

### 显示用户
app01/models.py
```py
from django.db import models

class UserInfo(models.Model):
    name = models.CharField(max_length=32)
    password = models.CharField(max_length=64)
    age = models.IntegerField(default=18)
```
查看数据库
```sql
mysql> desc app01_userinfo;
+----------+-------------+------+-----+---------+----------------+
| Field    | Type        | Null | Key | Default | Extra          |
+----------+-------------+------+-----+---------+----------------+
| id       | bigint      | NO   | PRI | NULL    | auto_increment |
| name     | varchar(32) | NO   |     | NULL    |                |
| password | varchar(64) | NO   |     | NULL    |                |
| age      | int         | NO   |     | NULL    |                |
+----------+-------------+------+-----+---------+----------------+
4 rows in set (0.00 sec)

mysql> select * from app01_userinfo;
+----+-------+----------+-----+
| id | name  | password | age |
+----+-------+----------+-----+
|  1 | peter | 123      |  18 |
|  2 | jack  | 123      |  18 |
+----+-------+----------+-----+
2 rows in set (0.00 sec)
```
显示用户列表
demo1/urls.py
```py
from app01 import views
urlpatterns = [
    path('user/info/', views.user_info),
]
```
app01/views.py
```py
def user_info(request):
    user_list = UserInfo.objects.all()
    return render(request, "user_info.html", {"user_list": user_list})
```
app01/templates/user_info.html
```html
<html lang="en">
<head><meta charset="UTF-8"><title>Title</title></head>
<body>
    <table border="1">
        <thead>
            <tr><th>ID</th><th>Name</th><th>Password</th><th>Age</th></tr>
        </thead>
        <tbody>
        {% for user in user_list %}
            <tr>
                <td>{{ user.id }}</td>
                <td>{{ user.name }}</td>
                <td>{{ user.password }}</td>
                <td>{{ user.age }}</td>
            </tr>
        {% endfor %}
        </tbody>
    </table>
</body>
</html>
```

### 添加用户
用户在页面的表单上输入用户信息, 再通过POST请求提交
demo1/urls.py
```py
urlpatterns = [
	# ...
    path('info/add/', views.user_add),
]
```

app01/views.py
```py
from app01.models import Department,UserInfo
def user_add(request):
    if request.method == "GET":
        return render(request, "user_add.html")

    username = request.POST.get("user")
    password = request.POST.get("password")
    age = request.POST.get("age")
    UserInfo.objects.create(name=username, password=password, age=age)
    return HttpResponse("ADD USER {} pwd: {} age: {} Done".format(username, password, age))
```

app01/templates/user_add.html
```
{% load static %}
<!DOCTYPE html>
<html lang="en">
<head><meta charset="UTF-8"><title>User Add</title></head>
<body>
    <form method="post"> <!-- action可以省略 -->
        {% csrf_token %}
        <input type="text" name="user", placeholder="username" />
        <input type="text" name="password", placeholder="password" />
        <input type="text" name="age", placeholder="age" />
        <input type="submit" value="submit" />
    </form>
</body>
</html>
```
注：如果POST提交地址和当前页面地址一样，可以省略action="/user/add/"

添加成功后自动跳转到用户页面
app01/views.py
```py
def user_add(request):
	# ...
    return redirect("/user/info")
```

在用户页面中支持新增用户的功能
```html
<a href="/user/add">Add User</a>
```

### 删除用户
demo1/urls.py
```py
from app01 import views
urlpatterns = [
    path('user/delete/', views.user_delete),
]
```

app01/views.py
```py
def user_delete(request):
    if request.method == "GET":
        return render(request, "user_delete.html")
    # POST
    username = request.POST.get("user")
    UserInfo.objects.filter(name=username).delete()
    return redirect("/user/info")
```

app01/templates/user_delete.html
```html
{% load static %}
<html lang="en">
<head><meta charset="UTF-8"><title>User Delete</title></head>
<body>
    <form method="post">
        {% csrf_token %}
        <input type="text" name="user", placeholder="username" />
        <input type="submit" value="submit" />
    </form>
</body>
</html>
```