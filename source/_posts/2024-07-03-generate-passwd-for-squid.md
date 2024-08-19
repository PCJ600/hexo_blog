---
layout: next
title: Squid配置用户名密码的方法
date: 2024-07-03 22:01:37
categories: Squid
tags: Squid
---

## 环境
Centos7.9 Squid 3.5.20

## 步骤
1 使用`htpasswd`工具，生成用户名密码。 例如这里添加用户名peter, 密码123.
```
yum install httpd-tools
htpasswd -c /etc/squid/squid_user peter
New password: 123
Re-type new password: 123
Adding password for user peter
```
检查密码文件`/etc/squid/squid_user`，可以找到刚才添加的用户`peter`
```
cat /etc/squid/squid_user
peter:$XXXXXXXXXXXXXXXXXXX
```
对密码文件设置适当权限
```
chown squid /etc/squid/squid_user
```
验证用户名和密码是否正确, 执行`basic_ncsa_auth`程序，输入`peter 123`，显示OK说明正确。
```
/usr/lib64/squid/basic_ncsa_auth /etc/squid/squid_user
peter 123
OK
```
<!-- more -->

2 修改squid配置文件`/etc/squid/squid.conf`，添加认证相关的配置
```
# Insert your own rules here to allow access from your clients

# http_access allow localhost 加注释，表示localhost也需要认证

auth_param basic program /usr/lib64/squid/basic_ncsa_auth /etc/squid/squid_user
auth_param basic children 5
auth_param basic realm Proxy Authentication Required
auth_param basic credentialsttl 2 hours
auth_param basic casesensitive on

acl authUsers proxy_auth REQUIRED
http_access allow authUsers

http_access deny all
```

修改配置完成后，重启Squid (`systemctl restart squid`)

## 验证
使用`curl`测试Squid用户名密码认证配置
不使用用户名密码认证，访问失败，返回407
![](image1.png)
使用正确的用户名密码认证，访问成功
![](image2.png)
## 参考
【1】[Squid设置用户名密码](https://www.cnblogs.com/blxt/p/14501176.html)
