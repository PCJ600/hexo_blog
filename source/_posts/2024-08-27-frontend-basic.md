---
layout: next
title: 2024-08-27-frontend-basic
date: 2024-08-27 22:33:26
categories: draft
tags: 
- frontend
- HTML
---

# Flask快速上手
FLask官方文档: [https://flask.palletsprojects.com/en/3.0.x/quickstart/#a-minimal-application](https://flask.palletsprojects.com/en/3.0.x/quickstart/#a-minimal-application)

## 第一个Flask App
Linux上安装Flask
```bash
pip3 install Flask
flask --version | grep Flask
Flask 3.0.3
```
写一个最小的Flask App, 添加web.py：
```py
from flask import Flask

app = Flask(__name__)
@app.route('/')
def index():
    return 'hello'

if __name__ == '__main__':
    app.run()
```
运行App
```bash
flask --app web run # 只允许本地访问, 端口号默认5000
flask --app web run --host=0.0.0.0 # 允许从外部访问
```

Flask提供render_template方法, 返回一个HTML页面
```py
@app.route('/show/info')
def index():
	# 默认从当前项目目录的templates文件夹中寻找index.html文件
	return render_template("index.html")
```

<!-- more -->

# HTML

## 浏览器可以识别的标签

### meta charset(编码)
```html
<head><meta charset="UTF-8"></head>
```
### title
```html
<head><title>MyTitle</title></head>
```

### 标题
```html
<h1>一级标题</h1>
<h6>六级标题</h6>
```

### div和span
```html
    <div>挖掘机技术</div>
    <div>找蓝翔</div>
    <span>厨师技术</span>
    <span>新东方</span>
```
页面效果
```
挖掘机技术
找蓝翔
厨师技术 新东方
```
标签可以划分为块级标签和行内标签
* div标签, 总是占了一整行, 叫【块级标签】
* span标签, 有多大占多大，一行可以多个，叫【行内标签】

### 超链接
```html
<a href="https://www.baidu.com">点击跳转百度</a>
<a href="/show/info">点击跳转内部网站</a>
```

### 图片
```html
<img src="图片地址"/>
```
显示自己的本地图片
```html
<img src="/static/image1.png" /> <!-- Flask项目中创建static目录, 图片放在static目录下-->
<img style="height: 100px;" src="/static/image1.png" /> <!-- 指定高度 -->
<img style="width: 20%" src="/static/image1.png" /> <!-- 指定宽度 -->
```
标签可以嵌套，例如实现点击图片跳转链接功能
```html
<a href="https://www.baidu.com" target="_blank">
	<img src="/static/image1.png" />
</a>
```

### 列表
```html
<!-- 项目编号是圆点 -->
<ul>
	<li>中国移动</li>
	<li>中国联通</li>
	<li>中国电信</li>
</ul>
<!-- 项目编号是数字(1,2,3) -->
<ol>
	<li>中国移动</li>
	<li>中国联通</li>
	<li>中国电信</li>
</ol>
```

### 表格
```html
<table border="1"> <!-- 加边框 -->
	<thead>
		<tr><th>ID</th><th>Name</th><th>Age</th></tr>
	</thead>
	<tbody>
		<tr><td>10</td><td>葫芦娃</td><td>19</td></tr>
		<tr><td>20</td><td>葫芦娃</td><td>19</td></tr>
		<tr><td>90</td><td>爷爷</td><td>91</td></tr>
	</tbody>
</table>
```

### INPUT
```html
<input type="text" />
<input type="password" />
<input type="file" />

<!-- 单选框, name设置成一致, 选项变为互斥 -->
<input type="radio" name="n1"/>male
<input type="radio" name="n1"/>female

<!-- 复选框 -->
<input type="checkbox">篮球
<input type="checkbox">足球
<input type="checkbox">乒乓球

 <!-- 普通按钮 -->
<input type="button" value="submit">
<!-- 提交表单-->
<input type="submit" value="submit">
```

### 下拉框
```html
<select>
	<option>北京</option>
	<option>上海</option>
	<option>南京</option>
</select>
```

### 多行文本
```html
<textarea rows="5"></textarea>
```

### form表单
HTML案例 —— 简单的注册表单页面:
新增注册页面 template/register.html
```html
<form method="post" action="/doreg">
    <div>
        用户名:<input type="text" name="user"/>
    </div>
    <div>
        密码:<input type="password" name="pwd"/>
    </div>
    <div>
    性别:
        <input type="radio" name="n1" value="1"/>male
        <input type="radio" name="n1" value="2"/>female
    </div>
    <div>
    爱好:
        <input type="checkbox" name="hobby" value="1">篮球
        <input type="checkbox" name="hobby" value="2">足球
    </div>
    <div>
    地域:
        <select name="city">
            <option value="bj">北京</option>
            <option value="sh">上海</option>
        </select>
    </div>
    <div>
        <select name="skill" multiple>
            <option value="1">吃饭</option>
            <option value="2">睡觉</option>
        </select>

    </div>
    <input type="submit", value="submit">
</form>
```
修改web.py, 内容如下:
```py
from flask import Flask, request

app = Flask(__name__)

@app.route('/register')
def register():
    return render_template('register.html')


@app.route('/doreg', methods=['POST'])
def doreg():
    user = request.form.get("user")
    pwd = request.form.get("pwd")
    gender = request.form.getlist("gender")
    hobby_list = request.form.getlist("hobby")
    city = request.form.get("city")
    skill_list = request.form.get("skill")
    print("user: ", user)
    print("pwd: ", pwd)
    print("gender: ", gender)
    print("city: ", city)
    print("skill_list: ", skill_list)
    return "register OK"
```

测试，浏览器中访问注册页面 https://{server_ip}/register 填写注册信息:
![](image1.png)

提交注册表单后，在flask日志中查看表单参数:
```
ImmutableMultiDict([('user', 'peter'), ('pwd', '123456'), ('gender', '1'), ('hobby', '1'), ('hobby', '2'), ('city', 'bj'), ('skill', '1'), ('skill', '2')])
user:  peter
pwd:  123456
gender:  ['1']
city:  bj
skill_list:  1
```

# CSS
用来"美化"标签

## CSS应用方式
1. 在head标签中添加style标签
```html
<head>
	<style>
		.c1{
			color:red;
		}
	</style>
</head>
<body>
	<h1 class="c1">标题</h1>
</body>
```

2. 写到CSS文件里
以Flask App为例，在static目录下创建test.css
```css
.c1{
	color:red;
}
.c2{
	color:blue;
}
```
templates目录下创建test.html
```html
<head>
    <link rel="stylesheet" href="/static/test.css" />
</head>
<body>
    <h1 class="c1">红标题</h1>
    <h1 class="c2">蓝标题</h1>
</body>
```
web.py下新增路由
```py
@app.route('/test', methods=['GET'])
def test():
    return render_template('test.html')
```

## CSS选择器
分类: ID选择器、类选择器、标签选择器、属性选择器、后代选择器

### 类选择器 (最常用)
```css
.cl{
	color: red;
}
```

### ID选择器
用的少, 因为id不能重复，只能修改一个标签的样式
```css
#c2{
	color: green;
}
```
### 标签选择器
为所有li标签定义样式
```css
li{
	color: blue;
}
```
### 属性选择器
在input标签中选择type='text'的标签
```css
input[type='text']{
	border: 2px solid pink;
}
```
在class为v1的标签中选择属性a=1的标签
```css
.v1[a="1"]{
	color: gold;
}
```
### 后代选择器
```css
.xx li{
	color: gold;
}
```

CSS选择器例子
```html
<html>
<head>
    <style>
        .c1{
            color: red;
        }
        #c2{
            color: green;
        }
        li{
            color: blue;
        }
		input[type="text"]{
			border: 2px solid pink;
		}
		.v1[a="1"]{
			color: red;
		}
		.xx li{
			color: gold;
		}
    </style>
</head>
<body>
	<!-- 类选择器-->
    <h1 class="c1">标题1</h1>
	<!-- ID选择器-->
    <h1 id="c2">标题2</h1>
	<!-- 标签选择器-->
    <ul>
        <li>移动</li>
        <li>联通</li>
    </ul>
	<!-- 属性选择器-->
	<input type="text">
	<div class="v1" a="0">V0</div>
	<div class="v1" a="1">V1</div>
	<!-- 后代选择器 -->
	<div class="xx">
		<li>上海</li>
		<li>北京</li>
	</div>
</body>
</html>
```
页面效果
![](image2.png)

## 样式
### 高度和宽度
```html
<style>
	.c1{
		height: 300px;
		width: 50%
	}
</style>
<body>
	<div class="c1">北京</div>
	<span class="c1">上海</span>
</body>
```
* 宽度支持百分比
* 对于行内标签, 设置高度和宽度默认无效
* 对于块级标签默认有效, 但右侧空白区域不能被其他标签利用

### display:inline-block
display:inline-block使标签同时具备块级和行内的特点
```html
<head>
    <style>
        .c1{
            display: inline-block;
            height: 100px;
            width: 300px;
            border: 1px solid red;
        }
    </style>
</head>
<body>
    <div class="c1">上海</div>
    <span class="c1">北京</div>
</body>
```
### display:block, display:inline
```html
<div style="display: inline;">上海</div>
<span style="display: block;">北京</span>
```
* display:block 可以把标签转为块级标签
* display:inline 可以把标签转为内联标签

Google浏览器 按F12 -> Elements -> 查看两个标签的宽和高，确认修改生效

### 字体
查HTML颜色代码网站： [HTML Color Codes](https://htmlcolorcodes.com/)
```html
<head>
<style>
	.c1 {
		color: #00FE7F; /* 颜色 */
		font-size: 44px; /* 字体大小 */
		font-weight: 500; /* 加粗 */
		font-family: Microsoft Yahei; /* 字体 */
	}
</style>
</head>
<div class="c1">上海</div>
```
* color指定颜色
* font-size指定字体大小
* font-weight指定加粗
* font-family指定字体

### 文字对齐方式
```html
<head>
	<style>
		.cl {
			height: 60px;
			width: 300px;
			border: 1px solid red;
			text-align: center; /* 水平居中 */
			line-height: 59px; /* 垂直居中 */
		}
	</style>
</head>
<div class="c1">北京</div>
```
* text-align: center 表示水平方向居中
* 垂直方向居中用line-height1


TO BE CONTINUED ...
