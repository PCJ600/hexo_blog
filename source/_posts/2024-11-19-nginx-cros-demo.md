---
layout: next
title: Nginx解决跨域问题的案例演示
date: 2024-11-19 22:19:49
categories: Nginx
tags: Nginx
---

介绍跨域问题前，首先了解浏览器的同源策略(Same-Origin Policy)

## 同源策略
同源策略是一种浏览器安全机制，限制了从一个源加载的文档或脚本与另一个源的资源进行交互的能力。

### 同源的定义
如果两个URL的协议、域名(IP)、端口号均相同，则它们是同源的，否则是跨源的。举例如下：
<!-- more -->
```
# http和https协议不同, 不满足同源
http://192.168.1.2/user/1
https://192.168.1.2/user/1

# IP不同，不满足同源
http://192.168.1.2/user/1
http://192.168.1.3/user/1

# 域名不同，不满足同源
http://www.zhangsan.com/user/1
http://www.lisi.com/user/1

# 端口不同，不满足同源
http://192.168.1.2/user/1
http://192.168.1.2:8080/user/1

# 协议、域名、端口均相同，满足同源, 默认端口就是80
http://www.zhangsan:80/user/1
http://www.zhangsan/user/1
```
### 同源策略的必要性
避免跨站点脚本攻击，防止恶意网站读取另一个网站的敏感数据，保护用户数据安全

## 跨域问题演示
有两台服务器A,B，A和B不满足同源策略。如果从A的页面发送异步请求到B获取数据，就会出现跨域问题。演示如下：

1、创建一台虚拟机, 用Nginx启动两个服务器，如下:
* 服务器A: http://192.168.52.200:80/a.html 页面上添加一个Button, 用户点击Button后发送异步请求到服务器B获取数据
* 服务器B: http://192.168.52.200:8080/user 返回一个JSON数据, 值为{"id": 1, "name": "peter", "age: 28"}

2、添加服务器A的页面a.html, 编辑`/var/www/server_a/a.html`，内容如下:
```html
<html>
    <head>
        <meta charset="utf-8">
        <script src="jquery-3.7.1.min.js"></script>
        <script>
            $(function(){
                $("#btn").click(function(){
                    $.get('http://192.168.52.200:8080/user', function(data){
                        alert(JSON.stringify(data));
                    });
                });
            });
        </script>
   </head>
   <body>
        <input type="button" value="Click Me, Get Data!" id="btn"/>
   </body>
</html>
```

3、修改Nginx.conf，添加两个服务器的配置
```
    # 服务器A
    server {
        listen       80;
        server_name  _;
        access_log   access.log;
        include /etc/nginx/default.d/*.conf;

        location / {
            root /var/www/server_a;
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }
    # 服务器B
    server {
        listen       8080;
        server_name  _;
        access_log   access.log;
        include /etc/nginx/default.d/*.conf;

        # 返回简单的JSON数据，用于演示
        location /user {
            default_type application/json;
            return 200 '{"id":1, "name":"peter","age":28}';
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }
```

4.重新加载Nginx并启动, 访问`192.168.52.200/a.html`，点击Button, 发现没有显示JSON数据，请求失败。浏览器按F12查到如下报错:
```
Access to XMLHttpRequest at 'http://192.168.52.200:8080/user' 
from origin 'http://192.168.52.200' has been blocked by CORS policy: 
No 'Access-Control-Allow-Origin' header is present on the requested resource.
```
这就是跨域问题。你在浏览器中直接访问`http://192.168.52.200:8080/user`，是可以返回数据的
```
http://192.168.52.200:8080/user
{"id":1, "name":"peter","age":28}
```

## 使用Nginx解决跨域问题
跨域问题涉及的几个响应头
* Access-Control-Allow-Origin 指定允许访问的源，可以配置多个，用逗号分隔, 也可以用*代表所有源
* Access-Control-Allow-Methods 指定允许的HTTP方法，值可以为GET POST PUT DELETE，用逗号分隔

**动手实现**
修改Nginx.conf, 使用add_header指令添加Header，如下:
```
    server {
        listen       8080;
        server_name  _;
        access_log   access.log;
        include /etc/nginx/default.d/*.conf;

        location /user {
			# 添加Header, 允许源地址为192.168.52.200的GET,POST请求
            add_header Access-Control-Allow-Origin http://192.168.52.200;	
			# add_header Access-Control-Allow-Origin *; #允许所有地址跨域访问
            add_header Access-Control-Allow-Methods GET,POST;
            default_type application/json;
            return 200 '{"id":1, "name":"peter","age":28}';
        }
    }
```
重新加载Nginx, 访问`192.168.52.200/a.html`，点击Button，发现请求成功。

## 参考
[https://www.cnblogs.com/javastack/p/16065851.html](https://www.cnblogs.com/javastack/p/16065851.html)
[https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS)