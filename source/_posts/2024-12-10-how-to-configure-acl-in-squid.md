---
layout: next
title: Squid ACL访问控制实践
date: 2024-12-10 19:59:48
categories: Squid
tags: Squid
---

## ACL简介
Squid ACL（Access Control List，访问控制列表）提供了代理控制的功能。 
通过设置ACL，Squid可以针对源地址、目标地址、访问的URL路径、访问的时间等各种条件进行过滤，从而实现对网络流量的精细控制。

基本的ACL元素语法:
```
acl name type value1 value2 ...
```
可以对一个ACL元素列举多个值，也可以多个ACL行使用同一个名字。例如，以下两种配置方式是等价的
```
acl safe_ports port 80 443 8080
```
等价于
```
acl safe_ports port 80
acl safe_ports port 443
acl safe_ports port 8080
```

## ACL使用方法实例
<!-- more -->
### 实例1：只允许网段10.206.216.0/24的客户机访问Squid, 拒绝其他客户端请求

修改squid.conf,添加如下几行配置
```
acl clients src 10.206.216.0/24
http_access allow clients
http_access deny all
```
使用如下命令校验配置文件，重启Squid
```
squid -k parse # 校验Squid配置文件
squid -k reconfigure # 重新加载配置
```

### 实例2：只允许访问指定的域名或IP
修改squid.conf,添加如下几行配置
```
acl SSL_ports port 443
http_access deny CONNECT !SSL_ports

acl allow_domain dstdomain www.baidu.com .google.com
acl allow_ip dst 180.101.50.188 142.250.199.100
http_access allow allow_domain
http_access allow allow_ip
```
注: 当ACL域名以"."开头，squid将它作为通配符，它匹配在该域的任何主机名，甚至域名自身。如果ACL域名不以"."开头，squid使用精确的字符串比较。

测试：
```
curl -x localhost:3128 https://www.baidu.com
curl -x localhost:3128 https://www.google.com
curl -x localhost:3128 180.101.50.188
curl -x localhost:3128 142.250.199.100
都应该返回200 OK

访问其他domain或IP, 返回403 Forbidden
curl -x localhost:3128 https://www.4399.com
curl: (56) Received HTTP code 403 from proxy after CONNECT
```

### 实例3：允许非443端口的FQDN通过
实际场景中，HTTPS server未必都是443标准端口。 例如: 我需要允许www.yourserver.com:8080这个FQDN通过，拒绝其他HTTPS请求，配置方法如下:
```
acl SSL_ports port 8080
http_access deny CONNECT !SSL_ports

acl PORT_8080 port 8080
acl allow_domain_8080 dstdomain www.yourserver.com
http_access allow allow_domain_8080 PORT_8080
```
测试:
```
curl -x peter:123@localhost:3128 https://www.yourserver.com:8080
返回200 OK，或者超时， 不会返回403 Forbidden
```

### 实例4：只允许通过认证的客户访问Squid
参考: [给Squid代理添加basic认证](https://blog.csdn.net/pcj_888/article/details/144328927)

### 一个综合案例
配置squid.conf，需求如下：
* 只允许通过HTTP basic认证的客户访问Squid
* 对于通过认证的客户, 只允许如下FQDN通过，拒绝其他的FQDN
	* www.baidu.com:443, *.google.com:443, 180.101.50.188:443 142.250.199.100:443, www.yourserver.com:8080

Squid.conf配置如下:
```
# INSERT YOUR OWN RULE(S) HERE TO ALLOW ACCESS FROM YOUR CLIENTS
#

auth_param basic program /usr/lib64/squid/basic_ncsa_auth /etc/squid/squid_user
auth_param basic children 5
auth_param basic realm Proxy Authentication Required
auth_param basic credentialsttl 2 hours
auth_param basic casesensitive on

acl authUsers proxy_auth REQUIRED

acl SSL_ports port 443
acl SSL_ports port 8080
http_access deny CONNECT !SSL_ports

acl PORT_443 port 443
acl PORT_8080 port 8080

acl allow_domain dstdomain www.baidu.com .google.com
acl allow_ip dst 180.101.50.188 142.250.199.100
http_access allow allow_domain PORT_443 authUsers
http_access allow allow_ip PORT_443 authUsers # 同时满足allow_ip, PORT_443, authUsers这三个ACL的请求才允许通过;

acl allow_domain_8080 dstdomain www.yourserver.com
http_access allow allow_domain_8080 PORT_8080 authUsers

http_access deny all
```

测试:
不带认证访问白名单的FQDN, 返回407
```
curl -x localhost:3128 https://www.baidu.com
curl: (56) Received HTTP code 407 from proxy after CONNECT
```
认证正确，但访问不在白名单的FQDN，返回403
```
curl -x peter:123@localhost:3128 https://www.4399.com
curl: (56) Received HTTP code 403 from proxy after CONNECT
curl -x peter:123@localhost:3128 https://www.yourserver.com
curl: (56) Received HTTP code 403 from proxy after CONNECT
```
认证正确，访问白名单的FQDN，返回200 OK
```
curl -x peter:123@localhost:3128 https://www.baidu.com
curl -x peter:123@localhost:3128 https://www.yourserver.com:8080
```

## 参考
【1】 [Squid中文权威指南 第6章](http://blog.zyan.cc/book/squid/chap06.html)
【2】 [巧用Squid的ACL和访问列表实现高效访问控制](https://developer.aliyun.com/article/415847)
