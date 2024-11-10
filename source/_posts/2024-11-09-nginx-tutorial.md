---
layout: next
title: Nginx基本使用
date: 2024-11-09 15:10:34
categories: Nginx
tags: Nginx
---

# Nginx简介
一个开源、高性能、高可靠的web服务器，支持热部署。占用内存少，并发能力强，使用BSD协议(允许免费使用或修改)。

# Nginx版本
* Nginx
* Nginx plus 商业版
* OpenResty
* Tengine

# Nginx使用场景
反向代理、虚拟主机、域名解析、负载均衡、防盗链、url重定向、https

# Nginx特点
* 高并发环境，比其他WEB服务器有更快响应
* 低内存消耗，单机支持10万以上并发连接
* 支持热部署
* BSD许可协议，允许用户免费使用Nginx，且允许用户在自己项目中直接使用或修改Nginx源码

# Nginx安装
[https://pcj600.github.io/2024/1109163902.html](https://pcj600.github.io/2024/1109163902.html)

<!-- more -->

# Nginx常用命令
```
nginx -h        # 查看Nginx用法
nginx -s reload # 向主进程发HUP信号，重新加载配置文件，热重启
nginx -s reopen # 向主进程发USR1信号，重新打开日志文件
nginx -s quit   # 向主进程发QUIT信号, 处理完所有正在进行请求后再停止服务 (优雅关闭)
nginx -s stop   # 向主进程发TERM信号，立即停止所有服务
nginx -T        # 查看当前配置
nginx -t        # 测试配置是否有问题
```

# Nginx主配置文件

主配置文件`/etc/nginx/nginx.conf`, 基础配置说明如下:
```
user nginx;                               # 以Nginx用户启动
worker_processes auto;                    # work process进程的个数
error_log /var/log/nginx/error.log;       # 错误日志路径
pid /run/nginx.pid; 

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;              # 单个进程可接收连接数
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 4096;

    include             /etc/nginx/mime.types;		# 服务器发送MIME type给客户端，让客户端知道请求的文件类型
    default_type        application/octet-stream;

    include /etc/nginx/conf.d/*.conf;

    server {
        listen       80;							# LISTEN端口
        listen       [::]:80;
        server_name  _;								# 主机名
        root         /usr/share/nginx/html;			# 在哪里找页面

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        error_page 404 /404.html;
        location = /404.html {
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }
}
```

# 虚拟主机配置
* 虚拟主机是指，在一台服务器主机虚拟出多台主机，每台虚拟主机都可以具有独立的域名，具有完整的服务器功能
* 同一台主机上的虚拟主机之间是完全独立的。从网站访问者来看，每一台虚拟主机和一台独立的主机完全一样。
* 利用虚拟主机，不必为每个要运行的网站提供一台单独的Nginx服务器或单独运行一组Nginx进程。虚拟主机提供了在同一台服务器、同一组Nginx进程上运行多个网站的功能。

## 实例：在一台机器上配置多台虚拟主机
例如，我需要实现如下效果:
* 访问80端口返回/www/www页面, 显示"this is www web site"
* 访问88端口返回/www/vod页面，显示"this is vod web site"

**操作步骤：**
配置多个站点
```
mkdir -p /www/vod /www/www
cat "welcome to vod site" > /www/vod/index.html
cat "welcome to www site" > /www/www/index.html
```
修改nginx.conf，添加两个server配置，如下
```
    # 虚拟主机1
    server {
        listen       80;
        server_name  _;
        root         /www/www;

        include /etc/nginx/default.d/*.conf;

        error_page 404 /404.html;
        location = /404.html {
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }
    # 虚拟主机2
    server {
        listen       88;			# 第二个虚拟主机使用88端口
        server_name  _;
        root         /www/vod;		# 访问/www/vod下的页面

        include /etc/nginx/default.d/*.conf;

        error_page 404 /404.html;
        location = /404.html {
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
    }
```
使用`nginx -s reload`重新加载配置，测试80,88两个端口可以访问
```
# curl localhost:88
this is vod web site
# curl localhost:80
this is www web site
```
注: 如遇到Nginx报错: bind() to 0.0.0.0:8088 failed (13: Permission denied), 可参考[link](https://pcj600.github.io/2024/1110141108.html)


# servername的多种匹配方式

## 同一个servername中匹配多个域名
例如，我需要通过不同的域名访问相同的页面, 实现如下效果：
* vod.petertest.com -> 访问/vww/vod 页面 -> 输出 this is vod site
* vod1.petertest.com -> 也访问/www/vod 页面 -> 输出 this is vod site

**操作步骤：**
nginx.conf中, 只需一个server配置项，就可以配多个servername, 不需要复制server配置块。修改方法如下:
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

## 通配符匹配多个servername
修改nginx.conf
```
    server {
        listen       80;
        server_name  *.petertest.com;
        root         /www/vod;
        include /etc/nginx/default.d/*.conf;
		# ...
    }
```

## 通配符结束匹配
例如，我需要实现如下匹配：
* vod.petertest.com 匹配到`/www/www`
* 其他vod.petertest.XXX 匹配到`/www/vod`

**操作步骤：**
修改Nginx.conf, 如下:
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
        root         /www/vod;
        include /etc/nginx/default.d/*.conf;
		# ...
    }
```

测试结果
```
# curl vod.petertest.com
this is www web site
# curl vod.petertest.io
this is vod web site
```

## 正则匹配
例如，我需要实现如下匹配：
* vod.petertest.com 匹配到`/www/www`
* 其他vod.petertest.XXX 匹配到`/www/vod` 

**操作步骤：**
修改Nginx.conf, 如下:
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
        server_name  ~^[0-9]+\.petertest\.com$;			# 通配符结束匹配
        root         /www/vod;
        include /etc/nginx/default.d/*.conf;
		# ...
    }
```
测试结果：
```
# curl vod.petertest.com
this is www web site
# curl 123.petertest.com
this is vod web site
```

# 基于域名的几种互联网需求解析
hosts泛解析 https://cloud.tencent.com/developer/article/1534150 (dnsmaxq) 本机DNS指向dnsmasq,dnsmasq做泛解析，把域名都解析到同一个IP


## 多用户二级域名需求(微博)
*.weibo.com -> Nginx -> 真正的业务服务器(拿到域名，解析出二级域名)
## 短网址
*.com/asdasjda12312 -> Nginx -> 真正的网址


# 反向代理
正向代理：你访问谷歌用的梯子
反向代理：Nginx代理了"目标服务器"，去和客户端交互, 目标服务器被隐藏，客户看不见。
client -> router(互联网) -> 企业机房网关(gateway) -> Nginx -> 目标服务器

URLWrite

# 负载均衡 
Nginx -> 服务器集群(任意一台提供的服务都是相同的)

**实战操作:** 
先准备三台RHEL9 VM机器，每台机器都安装Nginx
```
192.168.52.200 Nginx1 
192.168.52.201 Nginx2
192.168.52.202 Nginx3
```

## 场景1: 配置最简单的proxypass (客户请求Nginx1, Nginx1把所有请求转发到Nginx2)
Nginx1上配置proxypass, 指向Nginx2, 把请求Nginx1的流量转发到Nginx2, 返回"Welcome to Nginx2"
先修改Nginx1的nginx.conf
```
    server {
        listen       80;
        server_name  vod.petertest.com;
        include /etc/nginx/default.d/*.conf;
        location / {
            proxy_pass http://192.168.52.201;	# 配置proxy_pass
        }
		# ...
    }
```
再修改Nginx2的nginx.conf
```
    server {
        listen       80;
        server_name  _;
        root         /var/www/html;
        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;
    }
```
Nginx2添加一个主页
```
mkdir -p /var/www/html
cat "Welcome to Nginx2" > /var/www/html/index.html
```
测试
```
curl 192.168.52.200:80
Welcome to nginx2
```

## 场景2: 配置多个proxypass
请求Nginx1后, Nginx1通过RR算法把请求转发给Nginx2, Nginx3

**实战操作:**
修改Nginx1的配置文件
```
# 添加upstream，和server同级
upstream my_servers{
	server 192.168.52.201:80;
	server 192.168.52.202:80;
}
server {
    listen       80;
    server_name  _;
    location / {
		proxy_pass http://my_servers;
	}
    # Load configuration files for the default server block.
    include /etc/nginx/default.d/*.conf;
}
```
测试
```
# curl 192.168.52.200:80
Welcome to nginx2
# curl 192.168.52.200:80
Welcome to Nginx3
```

## 负载均衡策略
### 设置权重策略(weight)
修改Nginx1的配置文件，如下:
```
upstream my_servers {
    server 192.168.52.201:80 weight=4;
    server 192.168.52.202:80 weight=1;
}
```
测试
```
# curl 192.168.52.200:80
Welcome to nginx2
# curl 192.168.52.200:80
Welcome to nginx2
# curl 192.168.52.200:80
Welcome to Nginx3
# curl 192.168.52.200:80
Welcome to nginx2
# curl 192.168.52.200:80
Welcome to nginx2
```
### 让某台服务器下线(down)
修改Nginx1的配置文件，如下:
```
upstream my_servers {
    server 192.168.52.201:80 weight=4 down;
    server 192.168.52.202:80 weight=1;
}
```
测试: (Nginx2下线, 所有请求只转发到了Nginx3)
```
# curl 192.168.52.200:80
Welcome to Nginx3
# curl 192.168.52.200:80
Welcome to Nginx3
```
### 备用服务器(backup)
把一台服务器设为backup，是指正常情况Nginx不会把请求转到这台服务器，但如果其他服务器都出现故障，Nginx会把请求转到这台备用服务器

修改Nginx1的配置文件，如下:
```
upstream my_servers {
    server 192.168.52.201:80;
    server 192.168.52.202:80 backup;
}
```
测试, 设置Nginx3为backup后, Nginx1把请求只转发到了Nginx2；此时我手动stop Nginx2, Nginx1把请求都转发到了Nginx3
```
# curl 192.168.52.200:80
Welcome to nginx2
# curl 192.168.52.200:80
Welcome to nginx2

# 此时手动stop Nginx2 
# curl 192.168.52.200:80
Welcome to Nginx3
```

### 其他负载均衡策略(不常用)
| 策略 | 描述 |
| -- | -- |
| ip_hash | 根据客户端IP地址转发到同一台服务器，可以保持会话 |
| least_conn | 最少连接访问 |
| url_hash | 根据用户访问的url定向转发请求，需要三房插件 |
| fair | 根据后端服务器响应时间选择，需要三方插件 |
注: 
* 单纯使用RR算法的问题: 不能维持会话
* 一般不用ip_hash, 因为客户的IP地址经常会改变
* least_conn也不常用, 对于不同带宽配置的服务器, 用least_conn策略不合适;
* fair根据后端服务器响应时间选择, 会造成网络倾斜，并不常用
* url_hash也不常用，不能做到维持会话，比如注册和登录的url是不同的，可能会转发到不同的服务器





