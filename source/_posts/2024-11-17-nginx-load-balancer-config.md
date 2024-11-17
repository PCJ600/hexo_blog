---
layout: next
title: Nginx负载均衡示例
date: 2024-11-17 16:55:53
categories: Nginx
tags: Nginx
---

# 什么是负载均衡
负载均衡是一种网络流量分配技术, 其核心目的是将大量网络请求均匀分配到多个服务器，提高网络服务的可靠性。有如下作用：
* 避免单点故障，提高可用性
* 灵活的水平扩展，通过增加或减少服务器数量，提升扩展性；可以用多台便宜机器代替一台高性能机器，省钱
* 优化资源利用率，减少响应时间，提升用户体验
* 负载均衡器上支持过滤，阻挡不安全的请求，提高系统安全性
<!-- more -->

# 正向代理和反向代理
这里给出通俗解释，详细的定义可以查Google
* 正向代理：国内你无法访问Google，必须用梯子，你用的这个梯子就是正向代理。
* 反向代理：Nginx代理了真实服务器，去和客户端交互; 对于客户端来说, 真实服务器被Nginx隐藏，这个Nginx就是反向代理。

# 负载均衡示例
Nginx负载均衡流程图如下:
```
                                                   -> web服务器1
                                                 /
客户端 --请求--> Nginx(负载均衡器) --proxy_pass--  -> web服务器2
                                                 \
                                                   -> web服务器3
```
Nginx负载均衡策略有很多，这里演示默认的Round-Robin轮询策略

先准备三台RHEL9 VM机器，每台机器都安装Nginx，用于后面的演示:
```
192.168.52.200 Nginx1 
192.168.52.201 Nginx2
192.168.52.202 Nginx3
```

## 示例1: 最简单的proxy_pass配置
测试机器：
```
192.168.52.200 Nginx1 
192.168.52.201 Nginx2
```
需求: Nginx1作为负载均衡器, 客户请求Nginx1, Nginx1把所有请求转发到Nginx2

**具体步骤**
先修改Nginx1的`nginx.conf`
```
    server {
        listen       80;
        server_name  _;
        include /etc/nginx/default.d/*.conf;
        location / {
            proxy_pass http://192.168.52.201;	# 配置proxy_pass
        }
		# ...
    }
```
再修改Nginx2的`nginx.conf`
```
    server {
        listen       80;
        server_name  _;
        root         /var/www/html;
        # ...
    }
```
为Nginx2添加主页
```
mkdir -p /var/www/html
cat "Welcome to Nginx2" > /var/www/html/index.html
```
测试
```
curl 192.168.52.200:80
Welcome to nginx2
```

## 示例2: 配置多个proxypass
测试机器:
```
192.168.52.200 Nginx1 
192.168.52.201 Nginx2
192.168.52.202 Nginx3
```
需求: Nginx1作为负载均衡器, 客户请求Nginx1, Nginx1通过Round-Robin轮询策略把请求转发给Nginx2, Nginx3

**具体方法**
修改Nginx1的配置文件
```
# 添加upstream配置块，和server块同级
upstream my_servers{
	server 192.168.52.201:80;
	server 192.168.52.202:80;
}
server {
    listen       80;
    server_name  _;
    location / {
		proxy_pass http://my_servers;	# 配置proxy_pass
	}
    # Load configuration files for the default server block.
    include /etc/nginx/default.d/*.conf;
}
```
测试
```
# curl 192.168.52.200:80
Welcome to nginx2
# curl 192.168.52.200:80
Welcome to Nginx3
```

## 示例3: 设置权重策略(weight)
需求:
* Nginx1作为负载均衡器, 客户请求Nginx1, Nginx1通过Round-Robin轮询策略把请求转发给Nginx2, Nginx3; 
* 给Nginx2设置更大的权重, 设置为Nginx3的四倍

**具体方法**
修改Nginx1的配置文件，如下:
```
upstream my_servers {
    server 192.168.52.201:80 weight=4;
    server 192.168.52.202:80 weight=1;
}
```
测试
```
# curl 192.168.52.200:80
Welcome to nginx2
# curl 192.168.52.200:80
Welcome to nginx2
# curl 192.168.52.200:80
Welcome to Nginx3
# curl 192.168.52.200:80
Welcome to nginx2
# curl 192.168.52.200:80
Welcome to nginx2
```
平均下来，每5次请求有4次转发给Nginx2, 1次转发给Nginx3, 与我们的权重设置吻合

## 示例4: 让某台服务器下线(down)
down将服务器标记为不可用，该服务器不参与负载均衡<br/>

需求: 对Nginx2进行停机维护，标记为不可用

**具体方法**
修改Nginx1的配置文件，如下:
```
upstream my_servers {
    server 192.168.52.201:80 weight=4 down;
    server 192.168.52.202:80 weight=1;
}
```
测试: (Nginx2下线, 所有请求只转发到了Nginx3)
```
# curl 192.168.52.200:80
Welcome to Nginx3
# curl 192.168.52.200:80
Welcome to Nginx3
```

## 示例5: 备用服务器(backup)
backup将该服务器标记为备份服务器。仅当其他服务器都出现故障时，Nginx才会把请求转到这台备用服务器

需求: 把Nginx3标记为备份服务器

**具体方法**
修改Nginx1的配置文件，如下:
```
upstream my_servers {
    server 192.168.52.201:80;
    server 192.168.52.202:80 backup;
}
```
测试, 设置Nginx3为backup后, Nginx1把请求只转发到了Nginx2；此时我手动stop Nginx2, 继续请求，Nginx1把请求都转发到了Nginx3
```
# curl 192.168.52.200:80
Welcome to nginx2
# curl 192.168.52.200:80
Welcome to nginx2

# 此时手动stop Nginx2 
# curl 192.168.52.200:80
Welcome to Nginx3
```

### 其他的负载均衡策略
| 策略 | 描述 |
| -- | -- |
| ip_hash | 根据客户端IP地址转发到同一台服务器，可以保持会话 |
| least_conn | 最少连接访问 |
| url_hash | 根据URL分配，需三方插件 |
| fair | 根据后端服务器响应时间选择，需三方插件 |

说明:  
* 单纯使用RR算法存在一个问题：不能维持Session
* ip_hash可以将某个客户IP的请求通过哈希算法固定到同一台服务器，实现Session共享；但是实际场景中，客户的IP经常会改变，所以单纯用ip_hash的做法不常见
* least_conn把请求发给连接数较少的服务器，这个单纯用也不常见; 对于不同带宽配置的服务器, 单纯用least_conn策略不合适
* fair根据后端服务器响应时间选择, 会造成网络倾斜，单纯用fair也不多见
* url_hash也不常用，不能做到维持会话。比如客户注册和客户登录的URL是不同的，通过哈希算法会转发到不同的服务器

