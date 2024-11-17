---
layout: next
title: Nginx入门
date: 2024-11-09 15:10:34
categories: Nginx
tags: Nginx
---

# Nginx简介
高性能的web服务器, 反向代理服务器, 负载均衡器, HTTP缓存

# 竞品
* Apache - 老牌Web服务器, 重量级, 高并发差
* Lighttpd - 轻量级,高性能Web
* Tomcat, Jetty - Java, 重量级Web
* IIS - 基于Windows

# Nginx版本
* Nginx
* Nginx plus 商业版
* OpenResty
* Tengine

# Nginx应用场景
* http服务器(静态服务器)
* 虚拟主机（一台服务器虚拟多个网站)
* 反向代理、负载均衡
* 安全管理, API Gateway, 跨域问题, UrlRewrite

# Nginx特点/为什么用Nginx
* 响应快
* 高并发(单机支持10w+, 并发连接上限取决与内存)
* 高扩展性
* 低内存消耗
* 高可靠性(worker进程异常退出, master进程可以快速拉起新的worker进程提供服务)
* 热部署(不停止服务, 升级Nginx的可执行文件, 更新配置，更换日志文件)
* 配置简单
* 最自由的BSD许可协议

# Nginx安装
[https://pcj600.github.io/2024/1109163902.html](https://pcj600.github.io/2024/1109163902.html)

# 内核参数优化
* file-max: 表示进程可以同时打开的最大句柄数，这个参数直接限制最大并发连接数
* tcp_tw_reuse: 设为1, 允许将TIME-WAIT的socket重新用于新TCP连接
* tcp_fin_timeout: 表示服务器主动关闭连接时, socket保持在FIN_WAIT_2状态最大时间
* tcp_max_tw_buckets: 表示OS允许TIME_WAIT套接字数量的最大值, 如果超过这个数字, TIME_WAIT被立刻清除并打印告警, 过多的TIME_WAIT会导致Web服务器变慢
* tcp_syncookies: 解决TCP的SYN攻击

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
stop快速停止服务, worker进程和master进程收到信号后立刻跳出循环
quit优雅停止服务, 关闭监听端口,停止接收新连接，把当前连接处理完，最后退出进程

# Nginx进程间的关系
一个master进程管理多个worker进程, worker进程数和CPU核心数相等

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

# 虚拟主机配置(server_name配置)
[https://pcj600.github.io/2024/1116173059.html](https://pcj600.github.io/2024/1116173059.html)

## Nginx Location配置
[TODO]

# 基于域名的几种互联网需求解析
补充: hosts泛解析 https://cloud.tencent.com/developer/article/1534150 (dnsmaxq) 本机DNS指向dnsmasq,dnsmasq做泛解析，把域名都解析到同一个IP
## 多用户二级域名需求(微博)
*.weibo.com -> Nginx -> 真正的业务服务器(拿到域名，解析出二级域名)
## 短网址
*.com/asdasjda12312 -> Nginx -> 真正的网址


# 反向代理
正向代理：国内无法访问谷歌, 必须用梯子，这个梯子就是正向代理
反向代理：指Nginx代理了"目标服务器"，去和客户端交互; 对于客户端来说, 目标服务器被隐藏。

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

# 动静分离
TODO

# URLReWrite

## 一个最简单的URLReWrite例子
场景: 客户 -> 互联网 -> Nginx(网关服务器，反向代理/负载均衡器) -> 业务服务器
准备两台VM机器:
```
192.168.52.200 VM1 Nginx(80端口)
192.168.52.201 VM2 业务服务器(5000端口)
```
实现如下效果:
* 访问 http://192.168.52.200/2.html, Nginx将请求转发到http://192.168.52.201:5000/admin?page=2
* 浏览器上只能看到http://192.168.52.200/2.html, 参数admin?page=2被隐藏

步骤：
1、先在VM2上搭建一个简单的目标服务器(这里用Flask搭建)
安装Flask
```
yum install -y python3-pip
pip3 install Flask
```
添加web.py
```py
#!/usr/bin/env python3

from flask import Flask, request

app = Flask(__name__)
@app.route('/')
def index():
    return 'hello'

@app.route('/admin', methods=['GET'])
def show_admin():
    page = request.args.get('page', default=0, type=int)
    return f'You are on page {page}'

if __name__ == '__main__':
    app.run()
```
启动server
```
flask --app web run --host=0.0.0.0 # 默认端口5000
```

2. 再修改VM1上的Nginx配置文件，启动Nginx
```
    upstream my_servers {
        server 192.168.52.201:5000;

    }
    server {
        listen       80;
        server_name  _;
        include /etc/nginx/default.d/*.conf;
		
        location / {
            rewrite ^/([0-9]+).html$ /admin?page=$1 break;	# rewriteUrl
            proxy_pass http://my_servers;
        }
		# ignore error_page ...
    }
```

3. 测试URLReWrite OK
```
# curl 192.168.52.200/2.html
You are on page 2
# curl 192.168.52.200/admin?page=2
You are on page 2
# curl 192.168.52.201:5000/admin?page=2
You are on page 2
# curl 192.168.52.201:5000/2.html
404 Not Found
```

# 防盗链
判断referer字段，如果不是合法的referer，返回403
修改Nginx.conf
```
    server {
        listen       80;
        server_name  vod.petertest.com;
        include /etc/nginx/default.d/*.conf;
        location / {
            valid_referers none 192.168.52.200; # referer不对, 返回403
            if ($invalid_referer) {
                return 403;
            }
            root /www/vod;
        }
```
使用curl测试防盗链:
```
# 没有指定referer, 或referer为本机IP, 返回200 OK
[root@localhost ~]# curl -I 192.168.52.200/kuangsan.png
HTTP/1.1 200 OK
[root@localhost ~]# curl --referer "http://192.168.52.200" -I 192.168.52.200/kuangsan.png
HTTP/1.1 200 OK

# referer不合法, 返回403
[root@localhost ~]# curl --referer "http://4399.com" -I 192.168.52.200/kuangsan.png
HTTP/1.1 403 Forbidden
```

# 高可用场景及解决方案
问题: Nginx本身不可用怎么办
```
客户 -> 互联网 -> Nginx1(192.168.52.200)(主) -> Web Servers 
							|
				  Nginx2(192.168.52.201)(备) 
```
直接交换两台Nginx的IP很麻烦，会有IP冲突问题; 可以使用一个虚拟IP作为入口
```
客户 -> 互联网 -> Virtual IP -> 主Nginx IP -> 业务服务器
```
keepalived+Nginx实现高可用 https://www.cnblogs.com/youzhibing/p/7327342.html

# HTTPS
TODO: 自签一个证书，做一个HTTPS服务, 测试一下看看


# Nginx进阶(高并发网站技术架构实战)

Ingress-Controller
https://www.cnblogs.com/crazymakercircle/p/17052040.html
