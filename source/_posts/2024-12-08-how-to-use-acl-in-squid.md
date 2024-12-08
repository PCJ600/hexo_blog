---
layout: next
title: 给Squid代理添加HTTP basic认证
date: 2024-12-08 15:07:16
categories: Squid
tags: Squid
---

HTTP basic认证是一种简单的认证机制，要求用户在请求资源前提供有效的用户名和密码。

# 实例: 给Squid代理添加HTTP basic认证
要求: 只允许用户名为peter,密码为123的请求通过认证, 其他请求返回407(Proxy认证失败)

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

![basic_auth1.png](basic_auth1.png)

使用正确的用户名密码认证，访问成功

![basic_auth2.png](basic_auth2.png)

# 通过编写Auth程序，自定义灵活的认证策略
最基础的basic认证方式，不一定能满足实际使用需求，考虑如下需求：
对于请求的USER, 用SHA1算法加密, 明文为（USER + 一个固定key), 如果请求的PASSWORD和加密后的字符串一致，就认为认证通过，否则不通过。 如何实现这种自定义认证？
```
- Username: GUID
- Password: Digest = SHA1 ( GUID + KEY )
```

Squid支持自定义认证，只需要编写自定义认证程序，再修改Squid配置项auth_param即可，方法如下:
## 编写自定义认证程序
写一个Python脚本, 主流程是一个死循环, 每次从标准输入读取请求中的user,passwd, 再通过SHA1算法判断密码是否正确。 如密码正确输出OK, 否则输出ERR
```python
#!/usr/bin/env python3

import hashlib
import sys
import logging
logging.basicConfig(level=logging.DEBUG,
                    format='%(asctime)s %(levelname)s %(message)s',
                    filename='/var/log/squid/auth.log', filemode='a')

def sha1_encrypt(username):
    combined = username + "squid"
    return hashlib.sha1(combined.encode('utf-8')).hexdigest()

def authenticate(username, password):
    encrypted_password = sha1_encrypt(username)
    return encrypted_password == password

if __name__ == '__main__':
    while True:
        line = sys.stdin.readline()
        auth = line.split()
        user, passwd = auth[0], auth[1]
        logging.debug("Auth user={}, pwd={}".format(user, passwd))
        if authenticate(user, passwd):
            sys.stdout.write('OK\n')
        else:
            sys.stdout.write('ERR\n')
        sys.stdout.flush()
``` 
把Python脚本安装到/my_auth, 再修改squid.conf, 添加如下内容:
```
http_access allow localhost # 允许localhost直接访问，无需认证

auth_param basic program /my_auth # 自定义认证程序
auth_param basic key_extras "%>rd"
auth_param basic children 5
auth_param basic realm Please enter your device credential
auth_param basic credentialsttl 24 hours # 24小时内不需要再认证
auth_param basic casesensitive on # 大小写敏感
acl AuthUsers proxy_auth REQUIRED

http_access allow AuthUsers

http_access deny all # 只允许localhost请求, 或者认证通过的请求
```
注:
* 允许localhost不带认证直接访问
* 别的机器访问必须通过认证，否则返回407

**测试**
用户名和密码正确, 返回200
```
curl -x peter:47ae7e2352e92154c82669de2a99dd2091e60faa@192.168.52.202:3128 https://www.baidu.com
...
200 OK 
```
密码错误，返回407
```
curl -x peter123:123@192.168.52.202:3128 https://www.baidu.com
curl: (56) Received HTTP code 407 from proxy after CONNECT
```
调试认证程序, 可以查看日志`/var/log/squid/auth.log`

# 参考
[https://wiki.squid-cache.org/Features/Authentication](https://wiki.squid-cache.org/Features/Authentication)