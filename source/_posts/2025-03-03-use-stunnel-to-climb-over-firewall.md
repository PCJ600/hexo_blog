---
layout: next
title: Stunnel加密通信方案
date: 2025-03-03 21:50:09
categories: Stunnel
tags:
- Stunnel
- Project
---

# 问题描述
我们在企业客户网络中部署了一些On-Premises的服务网关。 服务网关上部署了Squid代理, 给客户本地的产品访问云端提供正向代理, 实现统一的访问控制。
这些本地产品访问云服务会有很多的FQDN; 但是有些客户的防火墙老旧, 不支持通配符匹配, 很容易出现防火墙配置错误导致网络不通的问题。
针对这部分客户, 我们设计了一个基于Stunnel的加密方案, 解决网络不通的问题, 同时大幅简化客户的防火墙配置, 提高用户体验。
以下介绍Stunnel加密通信方案

![](stunnel_design.png)

<!-- more -->

# 不加密行不行?

先看下问题

Client (192.168.52.200)
墙内代理 (192.168.52.204)

## 直接出 ( Client -> Squid -> Server )

[图]

客户端到代理
![](image_direct0.png)

代理到目标服务器
![](image_direct1.png)

很明显，这种情况下, 防火墙可以直接通过包的IP地址获得你访问的目标服务器, 从TLS握手报文的SNI也可以发现你访问的目标服务器.

**HTTP隧道原理**:


## 墙外Squid出 ( Squid -> Squid )

[图]

那墙外搭个Squid是不是就可以了呢? 这样目的IP就是墙外代理的IP, 防火墙就没法通过包的IP地址知道我访问的目标服务器了?

客户端到代理
![](image_no_stunnel0.png)

代理到目标服务器
![](image_no_stunnel1.png)

虽然正文是加密的, 但是HTTP CONNECT报文不加密(能看到目标服务器和代理用户名和密码), 可以从TLS握手报文中的SNI看到目标服务器

![](image_no_stunnel2.png)

防火墙还是能发现你访问的目标服务器, 而且代理用户名和密码在公网传输, 是非常不安全的


为什么不能是HTTPS代理呢?

# Stunnel加密通信原理
[补图]
 
# 动手搭建
墙内的主机(192.168.52.204)
墙外的云服务器(47.103.80.253)


## 先配置云服务器(Stunnel+Squid)

安装软件
```
yum install -y squid stunnel
```

### 配置Squid

#### Squid启用身份验证
设置用户名/密码认证
```
yum install -y httpd-tools
htpasswd -c /etc/squid/squid_user test
New password: qY8kd0Cf
Re-type new password: qY8kd0Cf
Adding password for user test
```
对密码文件设置适当权限, 再次验证用户名和密码是否正确
```
cat /etc/squid/squid_user
test:$XXXXXXXXXXXXXXXXXXX

/usr/lib64/squid/basic_ncsa_auth /etc/squid/squid_user 
test qY8kd0Cf
OK
```
Squid配置文件中添加认证相关配置
```
# Insert your own rules here to allow access from your clients

# http_access allow localhost  加注释，表示localhost也需要认证

auth_param basic program /usr/lib64/squid/basic_ncsa_auth /etc/squid/squid_user
auth_param basic children 5
auth_param basic realm Proxy Authentication Required
auth_param basic credentialsttl 2 hours
auth_param basic casesensitive on

acl authUsers proxy_auth REQUIRED
http_access allow authUsers

http_access deny all
```
重启Squid,测试一下
```
systemctl restart squid

curl -x localhost:3128 https://www.baidu.com
curl: (56) Received HTTP code 407 from proxy after CONNECT
# curl -x test:qY8kd0Cf@localhost:3128 https://www.baidu.com
200 OK
```

### 配置Stunnel
生成一个自签名证书
```
openssl req -new -x509 -days 365 -nodes -out /etc/stunnel/stunnel.pem -keyout /etc/stunnel/stunnel.pem
```
修改Stunnel配置文件、
编辑`/etc/stunnel/stunnel.conf`
```
cert = /etc/stunnel/stunnel.pem
client = no

[squid]
accept = 443
connect = 127.0.0.1:3128
```
启动Stunnel
```
systemctl restart stunnel
```

## 再配置客户端


### 配置Stunnel
编辑`/etc/stunnel/stunnel.conf`
```
debug=5
client = yes
verify=0
output=/var/log/stunnel.log
pid = /var/run/stunnel.pid
[upstream]
accept = 8081

connect=47.103.80.253:443
sslVersion=TLSv1.2
```
启动Stunnel, 使用curl测试
```
systemctl restart stunnel

curl -x 'test:qY8kd0Cf@localhost:8081' https://www.trendmicro.com -i
200 OK
```

### 配置Squid
修改Squid.conf, 所有流量都转发到本地Stunnel, 需要指定云端Proxy的用户名和密码
```
http_access allow localhost

never_direct allow all
cache_peer 127.0.0.1 parent 8081 0 no-query default login=test:qY8kd0Cf

http_access deny all
```
启动Squid, 使用curl测试
```
systemctl start squid

curl -x localhost:3128 https://www.trendmicro.com -i
```

## 测试墙内到墙外的数据是否加密

在客户端上通过墙内代理访问墙外云服务器, 用wireshark抓包
```
curl -x 'test:qY8kd0Cf@192.168.52.204:8081' https://www.trendmicro.com -i
```
客户端到代理的报文如下, 可以看到明文请求:
![](image_ok0.png)


墙内代理到墙外云服务器的报文如下:

![](image_ok1.png)

可以看到请求数据已经被加密, 防火墙只能看到Stunnel Server的FQDN, 看不到真正请求的目标服务器(www.trendmicro.com)

综上, 通过Stunnel加密方案, 客户防火墙不再需要配置所有的FQDN, 只需要允许[Stunnel Server]:443的包通过即可