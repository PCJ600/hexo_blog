---
layout: next
title: 基于Stunnel加密通信方案设计, 突破防火墙限制
date: 2025-03-03 21:50:09
categories: Stunnel
top: 101
tags:
- Stunnel
- Project
---

# 前言
在我们的业务场景中，客户终端需要访问部署在云端的各种服务，这些云端服务分布在不同的域名下。 例如, 一个典型的客户需要访问数十个甚至上百个不同的FQDN来满足他的业务需求
然而, 许多客户的防火墙不支持通配符的FQDN白名单配置, 这意味者他们必须手动配置每个具体的FQDN，这种配置非常麻烦，而且一旦配置错误会导致网络不通，严重影响用户体验, 增加运维负担
为了解决这个问题，我设计了一个基于Stunnel的加密通信方案, 以简化客户的防火墙配置:
* 我们在客户的网关设备上部署了Stunnel，用于将Squid代理的所有HTTP流量进行TLS加密
* Stunnel作为透明加密层，接收来自Squid的流量，并将其封装到TLS隧道中，将加密后流量发送给云端的一个Cloud Proxy.
* Cloud Proxy是一个HTTPS代理, 负责解密网关设备传来的TLS流量，将解密后的请求转发到真正的后端服务

这种方案下，客户只需在防火墙配置Cloud Proxy这一个FQDN即可，无需逐一配置所有目标服务的FQDN，这个方案大幅简化了客户的防火墙配置，同时有效减少了网络不通问题

# Stunnel是什么, 被设计用来解决什么问题
Stunnel是一款开源软件，其主要功能是为应用程序或服务提供TLS/SSL加密支持。 它的设计初衷是为了增强那些本身不支持加密功能的传统应用程序或服务的安全性
通过Stunnel, 可以在严格的防火墙规则下实现安全的加密通信，同时有效绕过可能存在的网络访问限制。

<!-- more -->

# 为什么要使用Stunnel ?

## 直接通过墙内的Squid代理出去为什么不行 ?

![](direct.png)

演示一下, 创建两台Linux虚拟机
* 客户端 (192.168.52.200)
* Squid代理 (192.168.52.204)

客户端通过Squid代理直接访问目标服务器`www.trendmicro.com`
```
curl -x 192.168.52.204:3128 https://www.trendmicro.com
```

wireshark抓包结果如下:

客户端和Squid代理之间报文
![](image_direct0.png)

Squid代理到目标服务器之间的报文
![](image_direct1.png)

从报文中可以看出, 防火墙可以直接识别目标服务器的IP, 这会导致连接被阻止.

## HTTP tunnel(HTTP隧道)的概念
HTTP tunnel是HTTP/1.1引入的功能, 常用于两台网络受限的机器之间建立网络连接。 客户端通过发送HTTP CONNECT请求与代理建立TCP隧道, 以访问HTTPS服务
根据上面抓包结果, 可以梳理出HTTP tunnel的工作过程:
```
 客户端                                 Squid(代理)                           目标服务器
   |                                       |                                      |
   | --- TCP connection                 -->|                                      |
   |                                       |                                      |
   | --- CONNECT www.trendmicro.com:443 -->|                                      |
   |                                       |                                      |
   |                                       | --- TCP connection                -->|
   |                                       |                                      |
   | <-- HTTP 200 OK                    ---|                                      |
   |     Connection Established            |                                      | 
   |                                       |                                      |   
    ========================== CONNECT tunnel Established ======================= |	
   |                                       |                                      |
   | --- TLS Application Data           -->|                                      |
   |                                       |                                      |
   |                                       | --- TLS Application Data          -->|
   |                                       |                                      |
   |                                       | <-- TLS Stream                    ---|
   |                                       |     HTTP 200 OK                      |
   |                                       |                                      |
   | <-- TLS Stream                     ---|                                      |
   |     HTTP 200 OK                       |                                      |
```
* 客户端和代理服务器建立TCP连接
* 然后, 客户端向代理发送明文的HTTP CONNECT请求, 告知代理需要连接的目标服务器
* 代理收到请求后, 和目标服务器建立TCP隧道连接, 并返回200 Connection Established给客户端, 告诉客户端隧道已经成功建立
* 隧道建立后, 代理仅负责转发数据而不解析或修改内容，确保了HTTPS数据的安全性，即使对于代理而言也无法解密TLS加密的内容

## 在墙外只部署HTTP代理行不行 ?
![](squid-outof-firewall.png)

不可以, 因为根据HTTP隧道的工作原理, 虽然HTTPS请求本身是加密的, 但是初始的HTTP CONNECT报文仍然是明文传输的。
在这个明文报文中，会暴露目标服务器的FQDN, 导致请求被客户的防火墙过滤, 造成网络不通。

以下搭建环境演示:
* 客户端 (192.168.52.200)
* 墙内代理 (192.168.52.204)
* 墙外代理(云服务器) (47.103.80.253)

wireshark抓包：

客户端和墙内Squid代理之间的报文
![](image_no_stunnel0.png)

墙内Squid到墙外Squid之间的报文
![](image_no_stunnel1.png)

观察抓包结果可以发现:
* 尽管正文部分是经过TLS加密的，但HTTP CONNECT请求是明文传输的，这意味着防火墙可以识别其中的目标服务器信息及代理认证信息（如用户名和密码), 此外，TLS握手过程中Client Hello报文中的SNI也会暴露目标服务器信息，因此这种方式仍然会被防火墙识别并阻止。
* 另外，代理认证信息在公网上传输也是非常不安全的做法

![](image_no_stunnel2.png)

# 基于Stunnel加密通信方案

![](my_design.png)

说明：
* 墙内部署Squid + Stunnel client, 墙外部署Stunnel Server + Squid, 客户终端的代理设置为墙内的Squid
* 通过这种方案, 客户防火墙不需要逐一指定目标服务器的FQDN, 只需允许通往Stunnel服务器（端口443）的流量即可。

接下来，通过实际搭建环境演示一下
* 墙内的主机(192.168.52.204), 部署 Squid Client + Stunnel Client
* 墙外的云服务器(47.103.80.253), 部署 Stunnel Server + Squid Server

## 配置墙外的云服务器(部署 Stunnel Server + Squid Server)

1. 安装软件
```
yum install -y squid stunnel
```

2. 配置Squid

* Squid启用basic认证
```
yum install -y httpd-tools
htpasswd -c /etc/squid/squid_user test
New password: qY8kd0Cf
Re-type new password: qY8kd0Cf
Adding password for user test
```

* 设置密码，并验证密码文件是否正确生成
```
cat /etc/squid/squid_user
test:$XXXXXXXXXXXXXXXXXXX

/usr/lib64/squid/basic_ncsa_auth /etc/squid/squid_user 
test qY8kd0Cf
OK
```

* 修改Squid配置文件（/etc/squid/squid.conf），添加以下认证相关配置
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

* 重启Squid服务并测试
```
systemctl restart squid

curl -x localhost:3128 https://www.baidu.com
curl: (56) Received HTTP code 407 from proxy after CONNECT
# curl -x test:qY8kd0Cf@localhost:3128 https://www.baidu.com
200 OK
```

3. 配置Stunnel

* 生成一个自签名证书
```
openssl req -new -x509 -days 365 -nodes -out /etc/stunnel/stunnel.pem -keyout /etc/stunnel/stunnel.pem
```

* 编辑Stunnel配置文件`/etc/stunnel/stunnel.conf`
```
cert = /etc/stunnel/stunnel.pem
client = no

[squid]
accept = 443
connect = 127.0.0.1:3128
```

* 启动Stunnel
```
systemctl restart stunnel
```

## 配置墙内的机器(部署 Squid + Stunnel Client)

1. 配置Stunnel
* 编辑Stunnel配置文件 `/etc/stunnel/stunnel.conf`
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

* 启动Stunnel, 使用curl测试
```
systemctl restart stunnel

curl -x 'test:qY8kd0Cf@localhost:8081' https://www.trendmicro.com -i
200 OK
```

2. 配置Squid
* 修改Squid.conf, 所有流量都转发到本地Stunnel, 并指定云端Proxy的用户名和密码
```
http_access allow localhost

never_direct allow all
cache_peer 127.0.0.1 parent 8081 0 no-query default login=test:qY8kd0Cf

http_access deny all
```

* 启动Squid, 并使用curl测试
```
systemctl start squid

curl -x localhost:3128 https://www.trendmicro.com -i
```

## 测试墙内到墙外的数据是否加密
为了验证数据是否被加密，在客户端通过墙内代理访问墙外云服务器，并使用Wireshark抓包分析; 在客户端执行以下命令
```
curl -x 'test:qY8kd0Cf@192.168.52.204:8081' https://www.trendmicro.com -i
```

客户端到代理的报文:
![](image_ok0.png)

墙内代理到墙外云服务器的报文:
![](image_ok1.png)

可以看到这一段的请求数据已经被加密, 防火墙只能看到Stunnel服务器的FQDN, 无法识别真正的目标服务器

## 为啥不直接用HTTPS代理 ?
客户虚拟设备是临时部署的, HTTPS代理需要管理和维护SSL/TLS证书, 增加了部署的复杂性
在客户内网中使用HTTP代理可以保证安全，所有客户都走HTTPS代理有性能损失, Stunnel加密方案只针对防火墙配置有问题的部分客户开放

## 方案的缺点是什么
Stunnel在处理大量TLS加密流量时会遇到性能瓶颈, 参考[stunnel performance data](https://www.stunnel.org/perf.html)
![](stunnel_benchmark.png)

我们的解决方法是水平扩展, 让客户多部署几台Stunnel, 以处理更多的终端
![](final.png)

## 参考
[squid + stunnel >> 跨越长城，科学上网！](https://www.hawu.me/operation/886)
