---
layout: next
title: 快速安装Nginx
date: 2024-11-09 16:39:02
categories: Nginx
tags: Nginx
---

## 安装环境
OS: RockyLinux 9.4

## 查看Nginx版本
```
# yum list | grep nginx
nginx.x86_64                                         1:1.20.1-16.el9_4.1                 appstream
nginx-all-modules.noarch                             1:1.20.1-16.el9_4.1                 appstream
nginx-core.x86_64                                    1:1.20.1-16.el9_4.1                 appstream
nginx-filesystem.noarch                              1:1.20.1-16.el9_4.1                 appstream
nginx-mod-http-image-filter.x86_64                   1:1.20.1-16.el9_4.1                 appstream
nginx-mod-http-perl.x86_64                           1:1.20.1-16.el9_4.1                 appstream
nginx-mod-http-xslt-filter.x86_64                    1:1.20.1-16.el9_4.1                 appstream
nginx-mod-mail.x86_64                                1:1.20.1-16.el9_4.1                 appstream
nginx-mod-stream.x86_64                              1:1.20.1-16.el9_4.1                 appstream
pcp-pmda-nginx.x86_64                                6.2.0-5.el9_4                       appstream
```

<!-- more -->

## 通过yum安装Nginx
```
yum install -y nginx
```
安装成功后，查看Nginx版本信息
```
# nginx -v
nginx version: nginx/1.20.1
```
## 查看Nginx的安装路径(主程序，配置文件的路径)
```
# rpm -qpl nginx
/usr/bin/nginx-upgrade
/usr/lib/systemd/system/nginx.service
/usr/share/man/man8/nginx.8.gz
/usr/share/nginx/html/index.html
......

# rpm -ql nginx-core
/etc/logrotate.d/nginx
/etc/nginx/nginx.conf
/usr/lib64/nginx/modules
/usr/sbin/nginx
/var/lib/nginx
/var/log/nginx
/var/log/nginx/access.log
/var/log/nginx/error.log
.....
```
注:
* /etc/nginx/ Nginx配置文件路径
* /usr/share/nginx/html/ 通常把静态文件放这里

## 运行Nginx
```
systemctl stop firewalld 	# 先关闭防火墙
systemctl start nginx		# 启动Nginx

ss -antp | grep nginx		# 查看80端口已经LISTEN
LISTEN    0      511           0.0.0.0:80           0.0.0.0:*
LISTEN    0      511              [::]:80              [::]:*
```
## 测试
`curl http://{your_ip}:80` 返回200 OK, 或者你在浏览器里访问IP能看到Nginx页面说明成功

## 参考
[https://www.nginx.org.cn/article/detail/545](https://www.nginx.org.cn/article/detail/545)
