---
layout: next
title: 配置Nginx自签名SSL证书，支持HTTPS
date: 2024-12-11 22:06:49
categories:
- Nginx
tags: 
- Nginx
- HTTP
---

## 配置Nginx自签名SSL证书的流程
* 生成一个SSL自签名证书
* 客户端机器信任这个自签名证书
* 修改RHEL服务器的Nginx配置
* 在客户机用curl测试HTTPS

## 生成一个SSL自签名证书
在RHEL服务器上, 用openssl命令生成一个自签名证书
```
openssl genrsa -out server.key 2048 #生成一个2048位的RSA私钥,保存到server.key
openssl req -x509 -new -nodes -key server.key -sha256 -days 3650 -out server.crt -subj "/C=CN/CN=www.petertest.com/O=MyRootCA" # 使用私钥server.key, 生成一个自签名的根证书server.crt, 有效期10年
```
注: `CN=www.petertest.com`需要换成你的服务器域名, 否则curl测试会报错
<!-- more -->

## 客户端机器信任这个自签名证书
将server.crt传到客户端Linux机器上。 在RHEL/Centos机器上, 需要把证书传到目录`/etc/pki/ca-trust/source/anchors/`，再用如下命令更新系统信任的CA证书列表
```
update-ca-trust
```
## 修改RHEL服务器的Nginx配置
修改`nginx.conf`, 如下：
```
    server {
        listen 443 ssl;
        server_name www.peter.com; # 设置域名

        access_log   access.log;
        include /etc/nginx/default.d/*.conf;

        ssl_certificate /server.crt; # 指定证书的绝对路径
        ssl_certificate_key /server.key; # 指定私钥的绝对路径

        location / {
            default_type text/html;
            return 200 '<h1>Welcome to Nginx</h1>';
        }
    }
```
再把证书文件server.crt和私钥文件server.key拷到服务器的根目录下

## 测试
在客户机用curl测试, 返回200 OK
```
curl https://www.petertest.com:443 -i
HTTP/1.1 200 OK
Server: nginx/1.20.1
Date: Wed, 11 Dec 2024 14:02:13 GMT
Content-Type: text/html
Content-Length: 25
Connection: keep-alive

<h1>Welcome to Nginx</h1>
```

## 参考
[https://cloud.tencent.com/developer/article/1743989](https://cloud.tencent.com/developer/article/1743989)