---
layout: next
title: Nginx虚拟主机配置
date: 2024-11-16 17:30:59
categories: Nginx
tags: Nginx
---

## 什么是虚拟主机

虚拟主机是指，把一台物理主机划分为多台虚拟主机，每台虚拟主机都可以是一个独立的网站，可以有独立的域名，具有完整的服务器功能。

Nginx提供虚拟主机功能，使我们不必为每个网站都提供一台Nginx服务器；只需运行一组Nginx进程，就可以运行多个域名不同网站

## 配置虚拟主机的方法
Nginx配置虚拟主机的方式有三种:
* 基于域名的虚拟主机(不同的域名，相同的IP，这种方式用的最多)
* 基于端口的虚拟主机(通过不同的端口号区分虚拟主机)
* 基于IP的虚拟主机 (不同域名, 不同的IP, 需要多个网络接口, 用的比较少，不演示)
<!-- more -->

## 基于域名配置虚拟主机
例: 在一台Linux服务器中配置两台Nginx虚拟主机，实现如下要求:
* 访问域名vod.petertest.com -> 返回服务器的/www/vod路径下的页面
* 访问域名aud.petertest.com -> 返回服务器的/www/aud路径下的页面

### 修改客户机的HOSTS文件，添加服务器IP和域名的映射
修改/etc/hosts文件，添加如下两行: (192.168.52.200是我的Linux服务器IP)
```
192.168.52.200 vod.petertest.com
192.168.52.200 aud.petertest.com
```

### 为两个虚拟主机准备静态页面
```
mkdir -p /www/vod /www/aud
echo "welcome to vod site" > /www/vod/index.html
echo "welcome to aud site" > /www/aud/index.html
```

### 修改Nginx.conf的server配置块
```
    # 虚拟主机1(vod.petertest.com)
    server {
        listen       80;
        server_name  vod.petertest.com; # 设置域名
		access_log   vod_petertest_access.log; # 设置日志路径
        include /etc/nginx/default.d/*.conf;

		location / {
		    root /www/vod;
	    }
        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }
    # 虚拟主机2(aud.petertest.com)
    server {
        listen       80;
        server_name  aud.petertest.com; # 设置域名
		access_log   aud_petertest_access.log; # 设置日志路径
        root         /www/aud; # 访问/www/vod下的页面
        include /etc/nginx/default.d/*.conf;
		location / {
		    root /www/aud;
		}

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }
```

### 重新加载Nginx配置，测试
使用`nginx -t`检查配置文件是否正确，使用`nginx -s reload`重新加载配置，测试结果如下:
```
# curl vod.petertest.com
welcome to vod site
# curl aud.petertest.com
welcome to aud site
```
注: 如遇到Nginx报错: bind() to 0.0.0.0:8088 failed (13: Permission denied), 参考: [link](https://pcj600.github.io/2024/1110141108.html)


## 基于端口配置虚拟主机
例: 在一台Linux服务器中配置两台Nginx虚拟主机，实现如下要求:
* 访问域名vod.petertest.com:80 -> 返回服务器的/www/vod路径下的页面
* 访问域名vod.petertest.com:88 -> 返回服务器的/www/aud路径下的页面

### 修改Nginx.conf的server配置块
配置方法和"基于域名配置虚拟主机"类似，只需要修改server_name和listen的端口号即可
```
    # 虚拟主机1(vod.petertest.com)
    server {
        listen       80;
        server_name  vod.petertest.com; # 设置域名
		access_log   vod_petertest_access.log;
        include /etc/nginx/default.d/*.conf;

		location / {
		    root /www/vod;
	    }
        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }
    # 虚拟主机2(aud.petertest.com)
    server {
        listen       88; # 设置端口
        server_name  vod.petertest.com; # 设置域名
		access_log   aud_petertest_access.log;
        root         /www/aud;
        include /etc/nginx/default.d/*.conf;
		location / {
		    root /www/aud;
		}

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }
```

### 测试
使用`nginx -t`检查配置文件是否正确，使用`nginx -s reload`重新加载配置，测试结果如下:
```
# curl vod.petertest.com
welcome to vod site
# curl vod.petertest.com:88
welcome to aud site
```

## Nginx server_name的其他配置案例
### 同一个server_name匹配多个域名
通过不同的域名访问相同的页面, 实现如下效果：
* 访问vod.petertest.com -> 返回/vww/vod 页面
* 访问vod1.petertest.com -> 也返回/www/vod 页面

方法：修改nginx.conf的server配置块, 一个server_name中配置多个域名，如下:
```
    server {
        listen       80;
        server_name  vod.petertest.com vod1.petertest.com;	# 在一个server中配置多个servername
        root         /www/vod;
        include /etc/nginx/default.d/*.conf;
		# ...
    }
```
测试结果:
```
# curl vod.petertest.com
this is vod web site
# curl vod1.petertest.com
this is vod web site
```
### 通配符匹配多个server_name
例如，访问`.petertest.com`结尾的域名，返回/www/vod页面

Nginx.conf配置如下：
```
    server {
        listen       80;
        server_name  *.petertest.com;	# 通配符匹配
        root         /www/vod;
        include /etc/nginx/default.d/*.conf;
		# ...
    }
```
### 通配符结束匹配
实现如下效果：
* 访问vod.petertest.com -> 返回/vww/vod 页面
* 访问vod.petertest.XXX -> 返回/www/aud 页面

Nginx.conf配置如下：
```
    server {
        listen       80;
        server_name  vod.petertest.com;
        root         /www/www;
        include /etc/nginx/default.d/*.conf;
		# ...
    }
    server {
        listen       80;
        server_name  vod.petertest.*;			# 通配符结束匹配
        root         /www/aud;
        include /etc/nginx/default.d/*.conf;
		# ...
    }
```
测试结果:
```
# curl vod.petertest.com
this is www web site
# curl vod.petertest.io
this is aud web site
```
### 正则匹配
实现如下效果：
* 访问vod.petertest.com -> 返回/vww/vod 页面
* 访问123.petertest.com -> 返回/www/aud 页面

Nginx.conf配置如下：
```
    server {
        listen       80;
        server_name  vod.petertest.com;
        root         /www/www;
        include /etc/nginx/default.d/*.conf;
		# ...
    }
    server {
        listen       80;
        server_name  ~^[0-9]+\.petertest\.com$;			# 正则匹配
        root         /www/vod;
        include /etc/nginx/default.d/*.conf;
		# ...
    }
```
测试结果:
```
# curl vod.petertest.com
this is www web site
# curl 123.petertest.com
this is aud web site
```

## 参考
[手把手教你配置Nginx的虚拟主机](https://juejin.cn/post/7096443628326748174)
