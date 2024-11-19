---
layout: next
title: Nginx文件下载服务器搭建
date: 2024-11-17 15:31:33
categories: Nginx
tags: Nginx
---


# Nginx文件下载服务器搭建
80端口启动下载服务器, 下载`/var/www/downloads`目录下的文件，`nginx.conf`如下：
```
server {
    listen 80;
    location /downloads/ {
        root /var/www/downloads;
        autoindex on; # 显示目录
        autoindex_localtime on;
    }
}
```

浏览器中访问，可以查到文件
```
http://192.168.52.200/
Index of /
../
1.txt     17-Nov-2024 14:52      0
2.txt     13-Nov-2024 21:44      0
```
<!-- more -->

## 添加登录认证
有时我们需要对用户身份进行认证。这里采用basic认证, 方法如下:<br/>
先生成密码文件，用户名admin, 密码123456
```
echo -n "admin:" > /etc/nginx/conf.d/.nginx_http_passwd
openssl passwd >> /etc/nginx/conf.d/.nginx_http_passwd
# 输入密码123456
```
查看密码文件
```
cat /etc/nginx/conf.d/.nginx_http_passwd
admin:$1$6ptGV8Fm$781jZKRgxV8wB.HtyTSdk.
```
修改nginx.conf，添加auth配置
```
server {
    listen       80;

    auth_basic "Restricted site";
    auth_basic_user_file /etc/nginx/conf.d/.nginx_http_passwd;
    # ...
}
```
重启Nginx，通过浏览器测试，此时会要求用户输入用户名和密码。

## 参考
[https://zhuleichina.github.io/2019/12/12/%E5%9F%BA%E4%BA%8Enginx%E7%9A%84%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0%E4%B8%8B%E8%BD%BD%E6%9C%8D%E5%8A%A1%E5%99%A8.html](https://zhuleichina.github.io/2019/12/12/%E5%9F%BA%E4%BA%8Enginx%E7%9A%84%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0%E4%B8%8B%E8%BD%BD%E6%9C%8D%E5%8A%A1%E5%99%A8.html)
