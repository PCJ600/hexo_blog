---
layout: next
title: Nginx入门
date: 2024-11-09 15:10:34
categories: Nginx
tags: Nginx
---

https://wiki.wgpsec.org/knowledge/web/same-origin-policy.html

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
# 虚拟主机配置
[https://pcj600.github.io/2024/1116173059.html](https://pcj600.github.io/2024/1116173059.html)
## Nginx Location配置
[https://pcj600.github.io/2024/1117141829.html](https://pcj600.github.io/2024/1117141829.html)
## 负载均衡/反向代理
[http://pcj600.github.io/2024/1117165553.html](http://pcj600.github.io/2024/1117165553.html)

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


# 基于域名的几种互联网需求解析
补充: hosts泛解析 https://cloud.tencent.com/developer/article/1534150 (dnsmaxq) 本机DNS指向dnsmasq,dnsmasq做泛解析，把域名都解析到同一个IP
## 多用户二级域名需求(微博)
*.weibo.com -> Nginx -> 真正的业务服务器(拿到域名，解析出二级域名)
## 短网址
*.com/asdasjda12312 -> Nginx -> 真正的网址

# Nginx浏览器缓存的概念
web缓存种类:
* 客户端缓存(浏览器缓存)
* 服务端缓存(Nginx/Redis/Memcached)

为了节约网络资源加速浏览，对用户最近请求过文档进行存储，再次请求这个页面时，浏览器就可以从本地磁盘显示文档，加速浏览。

# HTTP协议中与缓存相关的Header
Expires: 缓存过期的日期和时间
Cache-Control: 设置和缓存相关的配置信息
Last-Modified: 请求资源最后修改时间(服务端的时间)
ETag: 请求变量的实体标签的当前值，例如MD5
https://cloud.tencent.com/developer/article/2264687
https://harttle.land/2017/04/04/using-http-cache.html
https://blog.csdn.net/sunny_day_day/article/details/107993349

Etag重要性 https://www.cnblogs.com/52linux/archive/2012/04/26/2470865.html
![]()

* 强缓存(直接取本地，不发请求到后端)
* 弱缓存(问一下后端，后端判断无变化，返回304)

# Nginx缓存设置(TODO)
expires指令
控制HTTP应答中的"Expires"(1.0的配置，问题是服务端时间和客户端时间存在不一致)和"Cache-Control"
```
expires [modified] time 
expires epoch | max | off
```
no-cache弱缓存
* time为负数, Cache-Control值为no-cache, 如果为整数, Cache-Control值为max-age=time
* epoch指定Expires的值为'1, January, 1970,00:00:01 GMT', Cache-Control值为no-cache
* max指定Expires的值'31 December 2037 23:59:59 GMT', Cache-Control值为10年
* off默认不缓存


# Nginx跨域问题解决(TODO)
浏览器同源策略：协议,域名(IP),端口相同均为同源

什么是跨域问题：
两台服务器A,B，如果从服务器A页面发送异步请求到服务器B获取数据，如A和B不满足同源策略，就会出现跨域问题

案例
服务器A 静态页面,点击按钮，请求B
服务器B 返回JSON数据

https://www.bilibili.com/video/BV1ov41187bq?vd_source=d8559c2d87607be86810cd806158bb86&spm_id_from=333.788.player.switch&p=63
























# 静态资源防盗链(TODO)
资源盗链指内容不在自己服务器，而是通过技术手段，绕过别人限制将别人内容放到自己页面上最终显示给用户，盗取大网站流量，用别人的资源搭自己网站

HTTP Header Referer

浏览器向web请求时，一般会带上referer，来告诉浏览器此网页是从哪个链接跳转过来的
后台服务器可以根据Referer判断自己是否为受信任的网站，如果是则放行，不是可以拒绝访问
https://www.bilibili.com/video/BV1ov41187bq?vd_source=d8559c2d87607be86810cd806158bb86&spm_id_from=333.788.player.switch&p=65
直接访问可以，通过XX页面访问不行

更精细的控制: Nginx第三方模块ngx_http_accesskey_module

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
Openresty
https://www.cnblogs.com/crazymakercircle/p/17052040.html


# 内核参数优化
* file-max: 表示进程可以同时打开的最大句柄数，这个参数直接限制最大并发连接数
* tcp_tw_reuse: 设为1, 允许将TIME-WAIT的socket重新用于新TCP连接
* tcp_fin_timeout: 表示服务器主动关闭连接时, socket保持在FIN_WAIT_2状态最大时间
* tcp_max_tw_buckets: 表示OS允许TIME_WAIT套接字数量的最大值, 如果超过这个数字, TIME_WAIT被立刻清除并打印告警, 过多的TIME_WAIT会导致Web服务器变慢
* tcp_syncookies: 解决TCP的SYN攻击

https://juejin.cn/post/6844904144235413512