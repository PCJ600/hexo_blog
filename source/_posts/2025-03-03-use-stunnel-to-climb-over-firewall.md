---
layout: next
title: 一种基于Stunnel加密通信方案设计, 绕过防火墙
date: 2025-03-03 21:50:09
categories: Stunnel
tags:
- Stunnel
- Project
---

# 前言
我们在企业客户网络中部署了一些On-Premises的服务网关。 服务网关上部署了Squid(HTTP代理), 为客户的本地终端访问云端提供正向代理

本地终端访问云端有很多的FQDN(*.trendmicro.com)
对于一些拥有老旧防火墙的客户而言，他们的防火墙设备不支持基于FQDN的通配符匹配, 这意味着客户必须手动逐一配置每个FQDN，而任何遗漏都可能导致连接被防火墙阻挡，导致网络不通, 影响用户体验。

针对这一问题, 我们设计了一种基于Stunnel的解决方案, 通过对TCP流量加密, 创建加密通道, 绕过客户防火墙限制, 解决网络不通问题.
在Stunnel方案下, 客户无需逐一配置所有目标服务器的FQDN, 只需配置Stunnel服务器的FQDN即可, 从而极大地简化了客户的防火墙配置流程。

# Stunnel是什么, 被设计用来解决什么问题
Stunnel是一个开源软件，用于提供TLS/SSL加密服务. 它的设计初衷是为了增强那些本身不支持加密功能的传统应用程序或服务的安全性; 通过Stunnel可以绕过防火墙限制并实现加密通信

# 为什么需要Stunnel ?

## 从墙内Squid代理直接出去不行吗 ? 

[图]

这种显然是不行的, 防火墙根据从代理到目标服务器这段报文, DNS查询能够获取FQDN, 获得客户端真实访问的目标服务器IP 

举个例子, 创建两台虚拟机, 客户端通过Squid代理直接访问目标服务器(www.trendmicro.com)  
* 客户端 (192.168.52.200)
* Squid (192.168.52.204)

wireshark抓包

客户端和Squid代理之间报文
![](image_direct0.png)

Squid代理到目标服务器之间的报文
![](image_direct1.png)


HTTP隧道常用于两台网络受限的机器之间建立网络连接。 客户端通过HTTP CONNECT请求与代理建立隧道, 从而访问HTTPS, 流程:
* 客户端和代理服务器三次握手, 建立TCP连接
* 客户端发送HTTP CONNECT请求给代理, 告诉代理自己需要连接的目标服务器
* 代理收到请求后, 和目标服务器建立TCP连接
* 代理返回200 Connection Established给客户端, 告诉客户端整个隧道已经建立
* 隧道建立后, 代理服务器只负责在客户端和服务间之间转发数据，不解析或修改数据。这保证了 HTTPS 数据的安全性，即使代理也无法解密 TLS 加密的内容（因为代理不知道密钥, 没有服务器的私钥)

HTTP隧道流程图:


# 在墙外部署一个Squid, 从墙内Squid直接转发到墙外Squid不行吗 ?

墙外搭个Squid, 墙内Squid请求墙外Squid, 防火墙看到的包的目的IP就是墙外Squid的IP, 就没法知道我访问的目标服务器了?

[图]

这样也不行, 下面抓个包演示一下, 

Client (192.168.52.200)
墙内代理 (192.168.52.204)
墙外代理 (47.103.80.253)


客户端到代理
![](image_no_stunnel0.png)

代理到目标服务器
![](image_no_stunnel1.png)

可以看到, 虽然正文是加密的, 但是HTTP CONNECT报文不加密(能看到目标服务器和代理用户名和密码), 可以从TLS握手报文中的SNI看到目标服务器

![](image_no_stunnel2.png)

流程图:


防火墙还是能发现你访问的目标服务器, 而且还有一个问题, 墙内代理用户名和密码在公网明文传输, 这样是非常不安全的


# 为啥不直接用HTTPS代理 ?


# Stunnel加密通信方案
[补图]
 
 
 
说明：
 


以下动手搭建:
墙内的主机(192.168.52.204), (Squid + Stunnel Client)
墙外的云服务器(47.103.80.253) (Stunnel Server + Squid)


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


## 参考
