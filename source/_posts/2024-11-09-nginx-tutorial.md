---
layout: next
title: Nginx入门
date: 2024-11-09 15:10:34
categories: Nginx
tags: Nginx
---

https://www.nginx.org.cn/article/detail/545

# Nginx简介
一个开源、高性能、高可靠的web服务器，支持热部署。占用内存少，并发能力强，使用BSD协议(允许免费使用或修改)。

# Nginx使用场景
* 反向代理, 包括缓存, 负载均衡
* 静态资源服务
* API服务, OpenResty

# Nginx特点
* 高并发环境，比其他WEB服务器有更快响应
* 低内存消耗，单机支持10万以上并发连接
* 支持热部署
* BSD许可协议，允许用户免费使用Nginx，且允许用户在自己项目中直接使用或修改Nginx源码

# Nginx快速安装
TODO: add link

# 重要概念

## 简单请求和非简单请求
首先我们来了解一下简单请求和非简单请求，如果同时满足下面两个条件，就属于简单请求：

请求方法是 HEAD、GET、POST 三种之一；
HTTP 头信息不超过右边着几个字段：Accept、Accept-Language、Content-Language、Last-Event-ID
Content-Type 只限于三个值 application/x-www-form-urlencoded、multipart/form-data、text/plain；
凡是不同时满足这两个条件的，都属于非简单请求。

浏览器处理简单请求和非简单请求的方式不一样：(抓包)
对于简单请求，浏览器会在头信息中增加 Origin 字段后直接发出，Origin 字段用来说明，本次请求来自的哪个源（协议+域名+端口）。

如果服务器发现 Origin 指定的源不在许可范围内，服务器会返回一个正常的 HTTP 回应，浏览器取到回应之后发现回应的头信息中没有包含 Access-Control-Allow-Origin 字段，就抛出一个错误给 XHR 的 error 事件；

如果服务器发现 Origin 指定的域名在许可范围内，服务器返回的响应会多出几个 Access-Control- 开头的头信息字段。

非简单请求

非简单请求是那种对服务器有特殊要求的请求，比如请求方法是 PUT 或 DELETE，或 Content-Type 值为 application/json。浏览器会在正式通信之前，发送一次 HTTP 预检 OPTIONS 请求，先询问服务器，当前网页所在的域名是否在服务器的许可名单之中，以及可以使用哪些 HTTP 请求方法和头信息字段。只有得到肯定答复，浏览器才会发出正式的 XHR 请求，否则报错。

## 跨域
在浏览器上当前访问的网站向另一个网站发送请求获取数据的过程就是跨域请求。
跨域是浏览器的同源策略决定的，是一个重要的浏览器安全策略，用于限制一个 origin 的文档或者它加载的脚本与另一个源的资源进行交互，它能够帮助阻隔恶意文档，减少可能被攻击的媒介，可以使用 CORS 配置解除这个限制。
```
# 同源的例子
http://example.com/app1/index.html  # 只是路径不同
http://example.com/app2/index.html

http://Example.com:80  # 只是大小写差异
http://example.com

# 不同源的例子
http://example.com/app1   # 协议不同
https://example.com/app2

http://example.com        # host 不同
http://www.example.com
http://myapp.example.com

http://example.com        # 端口不同
http://example.com:8080
```
## 正向代理和反向代理
正向代理：你的浏览器无法直接访问谷哥，这时候可以通过一个代理服务器来帮助你访问谷哥，那么这个服务器就叫正向代理。
反向代理: 去饭店吃饭，可以点川菜、粤菜、江浙菜，饭店也分别有三个菜系的厨师, 但是你作为顾客不用管哪个厨师给你做的菜，只用点菜即可，小二将你菜单中的菜分配给不同的厨师来具体处理，那么这个小二就是反向代理服务器。

## 负载均衡

## 动静分离

<-- TO BE CONTINUED -->




