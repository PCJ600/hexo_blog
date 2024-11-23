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

# Nginx安装
[https://pcj600.github.io/2024/1109163902.html](https://pcj600.github.io/2024/1109163902.html)
# 虚拟主机配置
[https://pcj600.github.io/2024/1116173059.html](https://pcj600.github.io/2024/1116173059.html)
## Nginx Location配置
[https://pcj600.github.io/2024/1117141829.html](https://pcj600.github.io/2024/1117141829.html)
## 负载均衡/反向代理
[http://pcj600.github.io/2024/1117165553.html](http://pcj600.github.io/2024/1117165553.html)
## 跨域问题
[https://pcj600.github.io/2024/1119221949.html](https://pcj600.github.io/2024/1119221949.html)
## Nginx+keepalived高可用
[https://pcj600.github.io/2024/1123135607.html](https://pcj600.github.io/2024/1123135607.html)

## 静态资源防盗链
资源盗链指内容不在自己服务器，而是通过技术手段，绕过别人限制将别人内容放到自己页面上最终显示给用户，盗取大网站流量，用别人的资源搭自己网站
浏览器向web请求时，一般会带上referer，来告诉浏览器此网页是从哪个链接跳转过来的
后台服务器可以根据Referer判断自己是否为受信任的网站，如果是则放行，不是可以拒绝访问
更精细的控制: Nginx第三方模块ngx_http_accesskey_module

## 基于域名的几种互联网需求解析
hosts泛解析 https://cloud.tencent.com/developer/article/1534150 
(dnsmaxq) 本机DNS指向dnsmasq,dnsmasq做泛解析，把域名都解析到同一个IP
## 多用户二级域名需求(微博)
*.weibo.com -> Nginx -> 真正的业务服务器(拿到域名，解析出二级域名)
## 短网址
*.com/asdasjda12312 -> Nginx -> 真正的网址

## Nginx缓存
web缓存种类:
* 客户端缓存(浏览器缓存)
* 服务端缓存(Nginx/Redis/Memcached)

为了节约网络资源加速浏览，对用户最近请求过文档进行存储，再次请求这个页面时，浏览器就可以从本地磁盘显示文档，加速浏览。

### HTTP协议中与缓存相关的Header
Expires: 缓存过期的日期和时间
Cache-Control: 设置和缓存相关的配置信息
Last-Modified: 请求资源最后修改时间(服务端的时间)
ETag: 请求变量的实体标签的当前值，例如MD5
https://cloud.tencent.com/developer/article/2264687
https://harttle.land/2017/04/04/using-http-cache.html
https://blog.csdn.net/sunny_day_day/article/details/107993349
https://www.cnblogs.com/52linux/archive/2012/04/26/2470865.html
* 强缓存(直接取本地，不发请求到后端)
* 弱缓存(问一下后端，后端判断无变化，返回304)

### Nginx缓存设置(TODO)
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

## 动静分离
TODO

## URLReWrite
Rewrite是Nginx提供的一个重要基本功能，用于实现URL的重写
URLReWrite依赖于PCRE支持，在编译安装Nginx之前，需要安装PCRE库
Nginx使用ngx_http_rewrite_module模块解析并处理Rewrite

官方文档: http://nginx.org/en/docs/http/ngx_http_rewrite_module.html

### URLReWrite的应用场景
域名跳转
域名镜像
独立域名
目录自动添加
合并目录
防盗链的实现



## 独立域名
一个web项目有多个模块，每个模块可设置独立域名
```
http://search.itcast.com:81 # 商品搜索模块
http://item.itcast.com:82 # 商品详情模块
http://cart.itcast.com:83 # 访问商品购物车模块
```

```
server{
    listen 81;
	server_name search.itcast.com;
	rewrite ^(.*) http://www.itcast.cn/search$1;
}
server{
    listen 82;
	server_name item.itcast.com;
	rewrite ^(.*) http://www.itcast.cn/item$1;
}
server{
    listen 83;
	server_name cart.itcast.com;
	rewrite ^(.*) http://www.itcast.cn/cart$1;
}
```

## URL后自动添加
https://www.cnblogs.com/Nicholas0707/p/12210551.html
/hello  301永久重定向 再 /hello/ 200 OK
/hello/ 200 OK

http://192.168.52.200/hello 和 http://192.168.52.200/hello/ 区别:
如果URL最后不加斜杠, Nginx会自动做一个301的重定向, 重定向地址如下：
```
server_name_in_redirect on | off (0.8.48版本以前默认为on)
值为on, http://server_name/hello/;
值为off http://原URL中的域名/hello/;
```
如果值为on, 访问不带斜杠，比如http://192.168.52.200/hello，重定向到http://localhost/hello/会有问题!
(只有老版本有这个问题)

## 合并目录
SEO(搜索引擎排名靠前)

例如: 网站有一个资源文件路径: /server/11/22/33/44/20.html
不利于搜索引擎优化，客户也不好记


# Nginx安全控制和SSL加密介绍
(TODO)



# Nginx挂了怎么办，怎么实现高可用
https://www.cnblogs.com/crazymakercircle/p/15476154.html
问题: Nginx本身不可用怎么办

**双机热备方案**
在一台服务器提供服务, 另一台服务器作为备用。当一台服务器不可用另一台就会顶替上去










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