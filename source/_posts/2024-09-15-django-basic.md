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
	'default': {
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

## Demo: 员工管理系统

### 创建项目
创建Django项目和APP
```
django-admin startproject webproj
python3 manage.py startapp app01
```
注册APP
demo1/settings.py
```py
INSTALLED_APPS = [
    'django.contrib.admin',
	# ...
    'app01.apps.App01Config', # Add your app config here !
]
```
运行
```
python3 manage.py runserver 0.0.0.0:8000
```

### 设计表结构
```
# 部门表
id title
1  研发
2  销售

# 员工表
id name password age account create_time depart_id
1  Tony 123      18
2
```

思考题: 如果部门删除，员工表怎么处理？
* 如果部门删除，员工也要裁掉 (级联删除)
* 员工不裁掉，可以置空

实际开发中为什么大公司要禁用外键约束？https://developer.aliyun.com/article/1171702

models.py
```py
class Department(models.Model):
    """ 部门表 """
    title = models.CharField(max_length=32)
class UserInfo(models.Model):
    """ 员工表 """
    name = models.CharField(max_length=16)
    password = models.CharField(max_length=64)
    age = models.CharField()
    account = models.DecimalField(max_digits=10,decimal_places=2, default=0)
    create_time = models.DateTimeField()

    # 部门ID, 外键, 级联删除, 允许部门为空
    depart = models.ForeignKey(to="Department", to_field="id",null=True, blank=True, on_delete=models.CASCADE)

	gender_choices = (
        (1, "男"),
        (2, "女"),
    )
    gender = models.SmallIntegerField(verbose_name="gender", choices=gender_choices)
```

### MySQL生成数据库
```
mysql> create database gx_day16 DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
```
settings.py
```
DATABASES = {
	'default': {
		'ENGINE': 'django.db.backends.mysql',
		'NAME': 'gx_day16',
		'USER': 'root',
		'PASSWORD': 'XXX',
		'HOST': 'localhost',
		'PORT': 3306,
	}
}
```
根目录执行
```
python3 manage.py makemigrations
python3 manage.py migrate
```

### 创建静态文件和模板文件
app目录下创建static, templates, 引入bootstrap, js
```
static/
├── js
│   └── jquery-3.7.1.min.js
└── plugins
    └── bootstrap-3.4.1
```

### 新增页面 —— 部门列表
webproj/urls.py
```py
from app01 import views
urlpatterns = [
    path('depart/list/', views.depart_list),
]
```
app01/views.py
```py
def depart_list(request):
	return render(request, 'depart_list.html')
```

app01/templates/depart_list.html
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
    <!-- 导航 -->
    <nav class="navbar navbar-default">
      <div class="container-fluid">
        <!-- Brand and toggle get grouped for better mobile display -->
        <div class="navbar-header">
          <button type="button" class="navbar-toggle collapsed" data-toggle="collapse" data-target="#bs-example-navbar-collapse-1" aria-expanded="false">
            <span class="sr-only">Toggle navigation</span>
            <span class="icon-bar"></span>
            <span class="icon-bar"></span>
            <span class="icon-bar"></span>
          </button>
          <a class="navbar-brand" href="#">用户管理系统</a>
        </div>

        <!-- Collect the nav links, forms, and other content for toggling -->
        <div class="collapse navbar-collapse" id="bs-example-navbar-collapse-1">
          <ul class="nav navbar-nav">
            <li><a href="#">部门管理</a></li>
            <li><a href="#">用户管理</a></li>
          </ul>
          <ul class="nav navbar-nav navbar-right">
            <li><a href="#">登录</a></li>
            <li class="dropdown">
              <a href="#" class="dropdown-toggle" data-toggle="dropdown" role="button" aria-haspopup="true" aria-expanded="false">当前用户<span class="caret"></span></a>
              <ul class="dropdown-menu">
                <li><a href="#">个人资料</a></li>
                <li><a href="#">我的信息</a></li>
                <li><a href="#">注销</a></li>
                <li role="separator" class="divider"></li>
                <li><a href="#">Separated link</a></li>
              </ul>
            </li>
          </ul>
        </div>
      </div>
    </nav>

    <!-- 内容 -->
    <div class="container-fluid">
        <div style="margin-bottom: 18px">
            <a class="btn btn-primary" href="#">新建部门</a>
        </div>
    </div>

    <div class="panel panel-default">
        <!-- Default panel contents -->
        <div class="panel-heading">部门列表</div>
        <!-- Table -->
        <table class="table table-bordered">
            <thead>
                <tr>
                    <th>ID</th>
                    <th>Name</th>
                    <th>Operation</th>
                </tr>
            </thead>
            <tbody>
				{% for d in departs %}
                <tr>
                    <td>{{ d.id }}</td>
                    <td>{{ d.title }}</td>
                    <td>
                        <a class="btn btn-primary">Edit</a>
                        <a class="btn btn-danger">Delete</a>
                    </td>
                </tr>
				{% endfor %}
            </tbody>
        </table>
    </div>

    <script src="{% static 'js/jquery-3.7.1.min.js' %}"></script>
    <script src="{% static 'plugins/bootstrap-3.4.1/js/bootstrap.js' %}"></script>
</body>
</html>
```
页面效果:
![](image1.png)

### 新增页面 —— 添加部门
app01/templates/depart_list.html
```html
<div class="container-fluid">
        <div style="margin-bottom: 18px">
            <a class="btn btn-success" href="/depart/add/">
                <span class="glyphicon glyphicon-plus-sign" aria-hidden="true"></span>
                新建部门
            </a>
        </div>
    </div>
```
webproj/urls.py
```py
urlpatterns = [
    path('depart/add/', views.depart_add),
]
```

depart_add.html
```html
    <!-- 带标题的面板 -->
    <div class="panel panel-default">
      <div class="panel-heading">
        <h3 class="panel-title">添加部门</h3>
      </div>
      <div class="panel-body">
            <!-- 水平排列的表单 -->
            <form method="post">
              {% csrf_token %}
              <div class="form-group">
                <label>部门标题</label>
                <input type="text" class="form-control" name="title" placeholder="title">
              </div>
              <button type="submit" class="btn btn-default">Submit</button>
            </form>
      </div>
    </div>
```

app01/views.py
```py
from django.shortcuts import render, HttpResponse, redirect
def depart_add(request):
    if request.method == "GET":
        return render(request, "depart_add.html")
    # POST
    title = request.POST.get("title")
    models.Department.objects.create(title="title")
    return redirect("/depart/list/")
```
页面效果:
![](image2.png)

### 删除部门
webproj/urls.py
```py
urlpatterns = [
    path('depart/delete/', views.depart_delete),
]
```
app01/views.py
```py
def depart_delete(request):
    nid = request.GET.get("nid")
    models.Department.objects.filter(title=nid).delete()
    return redirect("/depart/list/")
```
app01/templates/depart_list.html
```html
            <tbody>
                {% for d in departs %}
                <tr>
                    <td>{{ d.id }}</td>
                    <td>{{ d.title }}</td>
                    <td>
                        <a class="btn btn-primary">Edit</a>
                        <a class="btn btn-danger" href="/depart/delete/?nid={{ d.id }}">Delete</a>
                    </td>
                </tr>
                {% endfor %}
            </tbody>
```

### 修改部门
效果: 点击Edit后，把部门的title带到输入框里。
webproj/urls.py
```py
urlpatterns = [
	# http://127.0.0.1:80000/depart/1/edit/
    path('depart/<int:nid>/edit/', views.depart_edit),
]
```
app01/views.py
```py
def depart_edit(request, nid):
    if request.method == "GET":
        depart = models.Department.objects.filter(id=nid).first()
        return render(request, "depart_edit.html", {"depart": depart})
    # POST
    title = request.POST.get('title')
    models.Department.objects.filter(id=nid).update(title=title)
    return redirect("/depart/list/")
```
depart_list.html
```html
    {% for d in departs %}
     <tr>
    <td>{{ d.id }}</td>
    <td>{{ d.title }}</td>
    <td>
        <a class="btn btn-primary" href="/depart/{{ obj.id }}/edit/">Edit</a>
    </td>
    </tr>
    {% endfor %}
```
depart_edit.html
```html
    <div class="panel panel-default">
      <div class="panel-heading">
        <h3 class="panel-title">编辑部门</h3>
      </div>
      <div class="panel-body">
            <!-- 水平排列的表单 -->
            <form method="post">
              {% csrf_token %}
              <div class="form-group">
                <label>部门标题</label>
                <input type="text" class="form-control" name="title" value="{{ depart.title }}">
              </div>
              <button type="submit" class="btn btn-default">Submit</button>
            </form>
      </div>
    </div>
```
页面效果
![](image3.png)

### 显示用户列表
多个HTML页面都用到了相同的导航栏，可以把相同的组件抽成模板(layout.html)
```html
{% block content %}-{% endblock %}
```
在新页面引用layout.html
```html
{% extends 'layout.html' %}

{% block content %}
	<h1>首页</h1>
{% endblock %}
```

### 显示用户列表
urls.py
```py
urlpatterns = [
	path('user/list/', views.user_list),
]
```

MySQL里加几条用户数据
```sql
mysql> desc app01_userinfo;
+-------------+---------------+------+-----+---------+----------------+
| Field       | Type          | Null | Key | Default | Extra          |
+-------------+---------------+------+-----+---------+----------------+
| id          | bigint        | NO   | PRI | NULL    | auto_increment |
| name        | varchar(16)   | NO   |     | NULL    |                |
| password    | varchar(64)   | NO   |     | NULL    |                |
| age         | int           | NO   |     | NULL    |                |
| account     | decimal(10,2) | NO   |     | NULL    |                |
| create_time | datetime(6)   | NO   |     | NULL    |                |
| gender      | smallint      | NO   |     | NULL    |                |
| depart_id   | bigint        | NO   | MUL | NULL    |                |
+-------------+---------------+------+-----+---------+----------------+
mysql> insert into app01_userinfo values(1, 'peter', '123456', 18, 100, '2024-09-21 11:31:00', 0, 1);
mysql> insert into app01_userinfo(name,password,age,account,create_time,gender,depart_id) 
values('peter', '123456', 18, 100, '2024-09-21 11:31:00', 0, 1);
```

views.py
```py
def user_list(request):
    if request.method == "GET":
        users = models.UserInfo.objects.all()
		for user in users:
			print(user.id, user.name, user.account, user.create_time.strftime("%Y-%m-%d"), user.gender)
			print(user.get_gender_display)
			print(user.depart) # 自动关联查询
        return render(request, "user_list.html", {"users": users})
```

templates/user_list.html
```html
{% load static %}
    <table class="table table-bordered">
	            <thead>
                <tr>
                    <th>ID</th>
                    <th>Name</th>
                    <th>Password</th>
                    <th>Age</th>
                    <th>Account</th>
                    <th>Createtime</th>
                    <th>Gender</th>
                    <th>Depart</th>
                    <th>Operation</th>
                </tr>
            </thead>
            <tbody>
                {% for u in users %}
                <tr>
                    <td>{{ u.id }}</td>
                    <td>{{ u.name }}</td>
                    <td>{{ u.password }}</td>
                    <td>{{ u.age }}</td>
                    <td>{{ u.account}}</td>
                    <td>{{ u.create_time | date:"Y-m-d H:i:s" }}</td>
                    <td>{{ u.get_gender_display }}</td>
                    <td>{{ u.depart.title }}</td>
                    <td>
                        <a class="btn btn-primary">Edit</a>
                        <a class="btn btn-danger">Delete</a>
                    </td>
                </tr>
                {% endfor %}
            </tbody>
    </table>
```

* 模板中解析datetime `<td>{{ u.create_time | date:"Y-m-d H:i:s" }}</td>`
* 模板中解析性别: `<td>{{ u.get_gender_display }}</td>`
* 关联查询部门: `<td>{{ u.depart.title }}</td>`


### 添加用户
原始方式存在的问题:
* 用户数据未做校验
* 如果输入错误，也没有错误提示
* 页面上，每一个字段都有重新写一遍
* 关联数据，需要手动获取传参，再展示到页面

为了解决以上问题，Django提供了ModelForm组件
urls.py
```py
urlpatterns = [
	path('user/add/', views.user_add),
]
```
views.py
```py
from django import forms
class UserModelForm(forms.ModelForm):
    class Meta:
        model = models.UserInfo
        fields = ["name", "password", "age", "account", "create_time", "gender", "depart"]

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        for name, field in self.fields.items():
            field.widget.attrs = {"class": "form-control"}

def user_add(request):
    if request.method == "GET":
        form = UserModelForm()
        return render(request, "user_add.html", {"form": form})
    # POST:
    form = UserModelForm(data=request.POST)
    if form.is_valid():
        form.save()
        return redirect("/user/list/")
    else:
        return HttpResponse(form.errors)
```
user_add.html
```html
<div class="panel-body">
    <!-- 水平排列的表单 -->
    <form method="post">
        {% csrf_token %}
        {% for field in form %}
            {{ field }}
        {% endfor %}
    </form>
</div>
```

### 编辑用户
urls.py
```py
urlpatterns = [
    path('user/<int:nid>/edit/', views.user_edit),
]
```
views.py
```py
def user_edit(request, nid):
    if request.method == "GET":
        obj = models.UserInfo.objects.filter(id=nid).first()
        form = UserModelForm(instance=obj)
        return render(request, "user_edit.html", {"form": form})

    # POST
    user = models.UserInfo.objects.filter(id=nid).first()
    form = UserModelForm(data=request.POST, instance=user)
    if form.is_valid():
        form.save()
        return redirect("/user/list/")
    else:
        return HttpResponse(form.errors)
```

### 删除用户
urls.py
```py
urlpatterns = [
    path('user/<int:nid>/edit/', views.user_delete),
]
```

views.py
```py
def user_delete(request, nid):
    obj = models.UserInfo.objects.filter(id=nid).delete()
    return redirect("/user/list/")
```

### 靓号管理
表结构
```sql
desc app01_prettynum;
+--------+-------------+------+-----+---------+----------------+
| Field  | Type        | Null | Key | Default | Extra          |
+--------+-------------+------+-----+---------+----------------+
| id     | bigint      | NO   | PRI | NULL    | auto_increment |
| mobile | varchar(11) | NO   |     | NULL    |                |
| price  | int         | NO   |     | NULL    |                |
| level  | smallint    | NO   |     | NULL    |                |
| status | smallint    | NO   |     | NULL    |                |
+--------+-------------+------+-----+---------+----------------+
5 rows in set (0.00 sec)
```

#### 展示靓号
models.py
```py
class PrettyNum(models.Model):
    """ 靓号表 """
    mobile = models.CharField(verbose_name="手机号", max_length=11)
    price = models.IntegerField(verbose_name="价格", default=0)
    level_choices = (
        (1, "1级"),
        (2, "2级"),
        (3, "3级"),
        (4, "4级"),
    )
    level = models.SmallIntegerField(verbose_name="级别", choices=level_choices, default=1)
    status_choices = (
        (1, "已占用"),
        (2, "未占用"),
    )
    status = models.SmallIntegerField(verbose_name="状态", choices=status_choices, default=2)
```
urls.py
```py
urlpatterns = [
    path('pretty/list/', views.pretty_list),
]
```
views.py
```py
from app01 import models
def pretty_list(request):
    prettys = models.PrettyNum.objects.all().order_by("-level")
    return render(request, "pretty_list.html", {"prettys": prettys})
```

### 新建靓号
urls.py
```py
urlpatterns = [
    path('pretty/add/', views.pretty_add),
]
```
views.py
```py
class PrettyModelForm(forms.ModelForm):
    class Meta:
        model = models.PrettyNum
        # fields = "__all__"
        # fields = ["mobile", "price", "level", "status"]
        exclude = ["level"]

def pretty_add(request):
    if request.method == "GET":
        form = PrettyModelForm()
        return render(request, "pretty_add.html", {"form": form})
    form = PrettyModelForm(data=request.POST)
    if form.is_valid():
        form.save()
        return redirect("/pretty/list/")
    else:
        return HttpResponse(form.errors)
```

对用户输入的格式做校验

### 编辑靓号
* path: /pretty/数字/edit/
* 使用ModelForm

urls.py
```py
urlpatterns = [
    path('pretty/<int:nid>/edit/', views.pretty_edit),
]
```

views.py
```py
def pretty_edit(request, nid):
    if request.method == "GET":
        pretty = models.PrettyNum.objects.filter(id=nid).first()
        form = PrettyModelForm(instance=pretty)
        return render(request, "pretty_edit.html", {"form": form})
    # POST
    pretty = models.PrettyNum.objects.filter(id=nid).first()
    form = PrettyModelForm(data=request.POST, instance=pretty)
    if form.is_valid():
        form.save()
        return redirect("/pretty/list/")
    else:
        return HttpResponse(form.errors)
```

pretty_edit.html
```html
{% extends 'layout.html' %}

{% block content %}
    <div class="container-fluid">
        <div style="margin-bottom: 18px">
            <a class="btn btn-success" href="/pretty/add/">
                <span class="glyphicon glyphicon-plus-sign" aria-hidden="true"></span>
                编辑靓号
            </a>
        </div>
    </div>

    <div class="panel panel-default">
        <!-- Default panel contents -->
        <div class="panel-heading">编辑靓号</div>
        <form method="post">
            {% csrf_token %}
            {% for field in form %}
            {{ field }}
            {% endfor %}
            <input type="submit" value="提交" />
        </form>
    </div>
{% endblock %}
```

### 查询手机号
数值搜索
```py
models.PrettyNum.objects.filter(id=12)
models.PrettyNum.objects.filter(id__gt=12)	# 大于12
models.PrettyNum.objects.filter(id__gte=12)	# 大于等于12
models.PrettyNum.objects.filter(id__lt=12)	# 小于12
models.PrettyNum.objects.filter(id__lte=12)	# 小于等于12
```

字符串搜索
```py
models.PrettyNum.objects.filter(mobile__startswith="1999")	# 开头
models.PrettyNum.objects.filter(mobile__endswith="999")		# 结尾
models.PrettyNum.objects.filter(mobile__contains="999")		# 包含
```

案例：加一个搜索框，显示所有匹配的手机号
```html
    <div style="float: right;width: 300px;">
        <form method="get">
      <div class="input-group">
        {% csrf_token %}
        <input type="text" class="form-control" name="search" placeholder="Search for...">
          <span class="input-group-btn">
          <input class="btn btn-default" type="submit">Go!</input>
          </span>
      </div>
        </form>
    </div>
```

views.py
```py
def pretty_list(request):
    if request.method == "GET":
        if request.GET.get("search"):
            search_mobile = request.GET.get("search")
            print(search_mobile)
            prettys = models.PrettyNum.objects.filter(mobile__contains=search_mobile)
            return render(request, "pretty_list.html", {"prettys": prettys})

        prettys = models.PrettyNum.objects.all()
        return render(request, "pretty_list.html", {"prettys": prettys})
```

### 分页显示靓号
效果：GET /pretty/list/?page=1 显示前10条记录
用切片
```
def pretty_list(request):
    if request.method == "GET":
        if request.GET.get("page"):
            page = int(request.GET.get("page"))
            begin = (page - 1) * 10
            end = page * 10
            prettys = models.PrettyNum.objects.all()[begin:end]
            return render(request, "pretty_list.html", {"prettys": prettys})
```

bootstrap上找一个分页组件
```html
<nav aria-label="Page navigation">
  <ul class="pagination">
    <li><a href="/pretty/list/?page=1">1</a></li>
    <li><a href="/pretty/list/?page=2">2</a></li>
    <li><a href="/pretty/list/?page=3">3</a></li>
  </ul>
</nav>
```

### 管理员操作
```py
class Admin(models.Model):
    """ 管理员 """
    username = models.CharField(verbose_name="username", max_length=32)
    password = models.CharField(verbose_name="password", max_length=64)
```

urls.py
```py
from app01 import views
urlpatterns = [
    path('admin/list/', views.admin_list),
    path('admin/add/', views.admin_add),
]
```
views.py
```py
def admin_list(request):
    if request.method == "GET":
        admins = models.Admin.objects.all()
        return render(request, "admin_list.html", {"admins": admins})

class AdminModelForm(forms.ModelForm):
    class Meta:
        model = models.Admin
        fields = "__all__"

def admin_add(request):
    if request.method == "GET":
        form = AdminModelForm()
        return render(request, "admin_add.html", {"form": form})
```

admin_list.html
```html
    <div class="panel panel-default">
        <!-- Default panel contents -->
        <div class="panel-heading">管理员列表</div>
        <!-- Table -->
        <table class="table table-bordered">
            <thead>
                <tr>
                    <th>ID</th>
                    <th>AdminUser</th>
                    <th>Password</th>
                    <th>Operation</th>
                </tr>
            </thead>
            <tbody>
                {% for obj in admins %}
                <tr>
                    <td>{{ obj.id }}</td>
                    <td>{{ obj.username }}</td>
                    <td>{{ obj.password }}</td>
                    <td>
                        <a class="btn btn-primary" href="">Edit</a>
                        <a class="btn btn-danger" href="">Delete</a>
                    </td>
                </tr>
                {% endfor %}
            </tbody>
        </table>
    </div>
```

admin_add.html
```html
{% extends 'layout.html' %}

{% block content %}
<!-- 带标题的面板 -->
    <div class="panel panel-default">
      <div class="panel-heading">
        <h3 class="panel-title">添加管理员</h3>
      </div>
      <div class="panel-body">
            <!-- 水平排列的表单 -->
            <form method="post">
              {% csrf_token %}

              {% for field in form %}
                <div class="form-group">
                    <label>{{ field.label }}</label>
                    {{ field }}
                </div>
              {% endfor %}
              <button type="submit" class="btn btn-primary">Submit</button>
            </form>
      </div>
    </div>
{% endblock %}
```

表单中添加确认密码, 判断两次密码输入一致
```py
from django.core.exceptions import ValidationError
class AdminModelForm(forms.ModelForm):
    confirm_password = forms.CharField(label="确认密码", widget=forms.PasswordInput)
    class Meta:
        model = models.Admin
        fields = ["username", "password", "confirm_password"]
        widgets = {
            "password": forms.PasswordInput
        }
    def clean_confirm_password(self):
        pwd = self.cleaned_data.get("password")
        confirm = self.cleaned_data.get("confirm_password")
        if confirm != pwd:
            raise ValidationError("Password Wrong")
        return confirm
```

### 编辑管理员
urls.py
```py
urlpatterns = [
    path('admin/<int:nid>/edit/', views.admin_edit),
]
```
views.py
```py
def admin_edit(request, nid):
    if request.method == "GET":
        admin = models.Admin.objects.filter(id=nid).first()
        form = AdminModelForm(instance=admin)
        return render(request, "admin_edit.html", {"form": form})

    admin = models.Admin.objects.filter(id=nid).first()
    form = AdminModelForm(data=request.POST, instance=admin)
    if form.is_valid():
        form.save()
        return redirect("/admin/list/")
    else:
        return HttpResponse(form.errors)
```

### 删除管理员
```py
urlpatterns = [
    path('admin/<int:nid>/delete/', views.admin_delete),
]
```
views.py
```py
def admin_delete(request, nid):
    if request.method == "GET":
        models.Admin.objects.filter(id=nid).delete()
        return redirect("/admin/list/")
```

### 用户认证(Session+Cookie认证)
Django默认把Session存到MySQL数据库中的django_session表里

先写一个登录页面，创建一个表单，包括username和password
校验用户名和密码输入正确，生成session到数据库, 跳转到/admin/list/页面
views.py
```py
def login(request):
    if request.method == "GET":
        form = LoginForm()
        return render(request, "login.html", {"form": form})
    # POST
    form = LoginForm(data=request.POST)
    if form.is_valid():
        admin_obj = models.Admin.objects.filter(**form.cleaned_data).first()
        if not admin_obj:
            form.add_error("password", "password or user error")
            return render(request, "login.html", {"form": form})

        # 保存Session
        request.session["info"] = {"id": admin_obj.id, "name": admin_obj.username}
        return redirect("/admin/list/")
    return HttpResponse(form.errors)
```

login.html
```html
<body>
<div class="account">
    <form method="post">
        {% csrf_token %}
        <div class="form-group">
            <label>Username</label>
            {{ form.username }}
            <span>{{ form.username.errors.0 }}</span>
        </div>
        <div class="form-group">
            <label>Password</label>
            {{ form.password }}
            <span>{{ form.password.errors.0 }}</span>
        </div>

        <input type="submit" value="login" class="btn btn-primary">
    </form>
</body>
```

### 数据库中查看Session
```sql
mysql> select * from django_session;
+----------------------------------+------------------------------------------------------------------------------------------------+----------------------------+
| session_key                      | session_data                                                                                   | expire_date                |
+----------------------------------+------------------------------------------------------------------------------------------------+----------------------------+
| o4ltz9wfdlh7w9p761wmyoedlwtk73t9 | eyJpbmZvIjp7ImlkIjoyLCJuYW1lIjoiSGVsbG8ifX0:1st6E1:wZE1TBMjag4FZ3dt-RA-9ObJPBs_G_j0vYsj6ixTA9Y | 2024-10-08 14:09:09.853533 |
+----------------------------------+------------------------------------------------------------------------------------------------+----------------------------+
```

### 鉴权操作(只有认证成功，才可以访问其他页面)
朴素的实现方式：
```py
def admin_list(request):
	# 如果没有session，跳转到登录页面
    info = request.session.get("info")
    if not info:
        return redirect("/login/")

    admins = models.Admin.objects.all()
    return render(request, "admin_list.html", {"admins": admins})
```

问题：所有视图都需要session认证，上面的实现太麻烦！

用Django中间件实现鉴权
app01/middleware/auth.py
```py
from django.utils.deprecation import MiddlewareMixin
from django.shortcuts import HttpResponse, redirect

class AuthMiddleWare(MiddlewareMixin):
    def process_request(self, request):
        # 注意：对于无需登录就应该访问的页面，不要做鉴权，否则会循环重定向
        if request.path_info == "/login/":
            return

        info_dict = request.session.get("info")
        print(info_dict)
        if info_dict:
            return
        return redirect("/login/")
		
    def process_response(self,request, response):
        print('M1 gone')
        return response
```
settings.py
```py
MIDDLEWARE = [
	'app01.middleware.auth.M1',
	'app01.middleware.auth.M2',
]
```