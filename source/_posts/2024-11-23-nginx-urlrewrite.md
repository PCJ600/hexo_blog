---
layout: next
title: Nginx URLRewrite案例演示
date: 2024-11-23 15:25:32
categories: Nginx
tags: Nginx
---

# URLRewrite
URL重写, 是Nginx中用于将请求重定向到其他URL的过程

# URLRewrite的应用场景
* 通过URL重写，企业可以使网站的URL更加规范和美观，提升用户体验。
* SEO优化，重写动态URL为静态URL，提高搜索引擎的收录效率和网站排名
* 域名迁移。 当企业更换域名时，通过URL重写将旧域名的访问永久重定向到新域名。(避免用户流失)
* 安全性增强, 隐藏实际文件路径, 防止攻击者直接访问文件
* 防盗链策略，防止其他网站直接链接到本站资源，减轻服务器负载
* URL重写保持用户的会话状态，实现会话同步

# URLRewrite案例演示
<!-- more -->

## 域名迁移
当企业更换域名时，通过URL重写将旧域名的访问永久重定向到新域名，避免用户流失。<br/>
例如访问京东，通过www.360buy.com, www.jingdong.com，最终都重定向到www.jd.com

案例演示:
访问www.petertest1.cn或www.petertest2.cn, 永久重定向到www.peter.com

### 创建一台Linux虚拟机, 准备三个域名
vim /etc/hosts
```
192.168.52.200 www.petertest1.cn
192.168.52.200 www.petertest2.cn
192.168.52.200 www.peter.com
```
注: 192.168.52.200是我的虚拟机IP

### 配置Nginx, 添加一个服务器, 配置重定向
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
        server_name www.petertest1.cn www.petertest2.cn;
        rewrite ^(.*) http://www.peter.com$1 permanent;
    }
```

### 测试
在浏览器访问`http://www.petertest1.cn/getUser`, 可以成功跳转到`http://www.peter.com` <br/>
且地址栏显示的是重定向后的URL: `http://www.peter.com/getUser`，测试如下：
```
http://www.peter.com/getUser
Welcome to peter
```
curl测试，返回301永久重定向, Location为`http://www.peter.com`
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

附: 301重定向抓包
先通过tcpdump抓包`tcpdump -i ens33 tcp and port not 22 -w output.pcap`, 再用Wireshark打开pcap
![](301.png)

## 参考
[https://blog.csdn.net/m0_62396418/article/details/135747521](https://blog.csdn.net/m0_62396418/article/details/135747521)
