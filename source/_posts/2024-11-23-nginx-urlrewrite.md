---
layout: next
title: Nginx URLRewrite案例演示
date: 2024-11-23 15:25:32
categories: Nginx
tags: Nginx
---

# URLRewrite是什么
URL重写, 是Nginx中用于将请求重定向到其他URL的过程

# URLRewrite的应用场景
* 域名迁移。 当企业更换域名时，通过URL重写将旧域名的访问永久重定向到新域名，避免用户流失。
* URL规范化。通过URL重写，使URL更加规范和美观，提升用户体验。
* 伪静态。重写动态URL为静态URL，提高搜索引擎的收录效率和网站排名
* 安全性增强。隐藏真实文件目录, 防止攻击者直接访问文件

# URLRewrite案例演示
<!-- more -->

## 案例1: 域名迁移
当企业更换域名时，通过URL重写将旧域名的访问永久重定向到新域名，避免用户流失。
例如访问京东，通过`www.360buy.com`, `www.jingdong.com`，最终都重定向到`www.jd.com`

**案例演示**
访问`www.petertest1.cn`或`www.petertest2.cn`, 永久重定向到`www.peter.com`

### 创建一台Linux虚拟机, 准备三个域名
修改虚拟机的/etc/hosts文件，内容如下:
```
192.168.52.200 www.petertest1.cn
192.168.52.200 www.petertest2.cn
192.168.52.200 www.peter.com
```
注: 192.168.52.200是我的虚拟机IP

### 配置Nginx, 添加一个服务器, 配置重定向
修改nginx.conf
```
    server {
        listen       80;
        server_name  www.peter.com;
        access_log   access.log;
        include /etc/nginx/default.d/*.conf;
        location / {
            default_type text/html;
            return 200 '<h1>Welcome to peter</h1>';
        }
    }
    server {
        listen 80;
        server_name www.petertest1.cn www.petertest2.cn; # 需要永久重定向的旧的域名
        rewrite ^(.*) http://www.peter.com$1 permanent; # rewrite实现重定向, 指定permanent返回301永久重定向, 不指定返回302临时重定向
    }
```

### 测试
在浏览器访问`http://www.petertest1.cn/getUser`, 可以成功跳转到`http://www.peter.com/getUser`，且地址栏显示的也是重定向后的URL: `http://www.peter.com/getUser`
再用curl测试，返回301永久重定向, Location为`http://www.peter.com`
```
curl http://www.petertest2.cn -i
HTTP/1.1 301 Moved Permanently
Server: nginx/1.20.1
Date: Sat, 23 Nov 2024 07:05:41 GMT
Content-Type: text/html
Content-Length: 169
Connection: keep-alive
Location: http://www.peter.com/
```

另一种实现域名永久重定向的配置方法(不使用URLRewrite)
```
    server {
        listen       80;
        server_name  www.peter.com;
        access_log   access.log;
        include /etc/nginx/default.d/*.conf;
        location / {
            default_type text/html;
            return 200 '<h1>Welcome to peter</h1>';
        }
    }
    server {
        listen 80;
        server_name example.com;
        return 301 http://www.peter.com$request_uri;
    }
```

附: 301永久重定向抓包
先通过tcpdump抓包`tcpdump -i ens33 tcp and port not 22 -w output.pcap`, 再用Wireshark分析

![](301.png)

这里可以看出重定向和请求转发的区别：客户端收到301后，客户端会再发起一次新的HTTP请求；而内部转发是服务器行为，客户端感知不到。

## 案例2: 伪静态
伪静态是指，把动态URL转换为静态URL的过程，以对用户更友好的方式显示在浏览器地址栏中。

**案例演示**
假设有一个动态URL`http://peter.com/page.php?id=1`, 我想重写成伪静态URL`http://peter.com/page/1`，Nginx中如何实现?

### 配置Nginx
修改nginx.conf
```
    server {
        listen       80;
        server_name  www.peter.com;
        access_log   access.log;
        include /etc/nginx/default.d/*.conf;
        location / {
            default_type text/html;
            return 200 '<h1>query_string: $query_string</h1>';
        }
        location ~ /page/(\d+)$ {
            rewrite ^/page/(\d+)$ /page.php?id=$1 last;
        }
    }
```
说明：
* 这里的rewrite语句最后要指定last，不用break。因为在这个案例中, 我们需要重写后的URL(`http://peter.com/page/1`)继续从头开始匹配所有的location，命中`location /`规则。(如果指定break会返回404，可自行尝试)

测试结果:
```
# curl localhost/page/1
<h1>query_string: id=1</h1>
# curl localhost/page/234
<h1>query_string: id=234</h1>
```

## 参考
【1】[Nginx rewrite地址重写](https://blog.csdn.net/m0_62396418/article/details/135747521)
【2】[Nginx-URLRewrite伪静态](https://developer.aliyun.com/article/1499869)