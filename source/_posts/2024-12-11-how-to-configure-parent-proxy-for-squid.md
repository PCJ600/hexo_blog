---
layout: next
title: 如何给Squid配置父代理，访问外部网络
date: 2024-12-11 21:10:31
tags:
---

## 需求
在公司局域网部署的Squid代理服务器，和局域网中其他电脑一样无法访问外网。 需要给Squid配置一个父级的HTTP代理，这个父代理能访问外网，这样Squid就具备了访问外网的能力。

## 基础知识
Squid提供了`cache_peer`指令, 以定义邻居cache, 并告知Squid如何与邻居通信。 
邻居cache有两种关系: 父子或姐妹
* 父子cache, 可用于配置父代理来请求内容。 
* 邻居cache, 用于提供额外的cache命中，提升性能。 例如某些请求cache丢失，在邻居cache有可能命中，这样比从原始服务器请求速度要快。

<!-- more -->
![image1.png](image1.png)


## 给Squid配置父代理的实例
创建两台Linux虚拟机, 每台机器部署Squid, 我使用的发行版是Rocky Linux 9.4

```
10.206.216.96 Squid服务器
10.206.216.93 父代理(可以任意搭, 我这里也用的Squid)
```
部署Squid方法参考: [https://blog.csdn.net/pcj_888/article/details/143024481](https://blog.csdn.net/pcj_888/article/details/143024481)

要求:
* 所有对内网的请求（10.206.216.0/24)直接走原始服务器，不经过父代理。
* 其他所有HTTP请求都必须转发到父代理，不允许Squid与原始服务器直接会话。

```
client -> Squid(10.206.216.96) -> 父代理(10.206.216.93) -> Internet

client -> Squid(10.206.216.96) -> LAN(10.206.216.0/24)
```

修改Squid服务器的主配置文件`squid.conf`, 添加如下内容：
```
http_access allow localhost
http_access allow localnet

acl direct-out-servers dst 10.206.216.0/24

cache_peer 10.206.216.93 parent 3128 0 no-query default
cache_peer_access 10.206.216.93 allow !direct-out-servers
cache_peer_access 10.206.216.93 deny all

never_direct allow all
```

解释:
* cache_peer定义邻居cache, 这里需要指定父代理的HOST和PORT
* cache_peer_access定义邻居cache的访问列表, 决定哪些请求允许或不允许发送到邻居cache.
* never_direct定义不需要直接发送到原始服务器的访问列表，当请求匹配该列表时，必须被发送到邻居cache

更详细的配置说明参考这篇: [Squid中文权威指南](http://blog.zyan.cc/book/squid/chap10.html#a0)

如果父代理需要basic认证, 可以修改cache_peer配置项, 添加认证信息
```
cache_peer 10.206.216.93 parent 3128 0 no-query default login=peter:123 # 例如，父代理user为peter, password为123
```

如果父代理挂了，允许Squid直接与原始服务器会话，去掉`never_direct allow all`这行配置即可。

校验并重新加载配置
```
squid -k parse
squid -k reconfigure
```
## 测试
1、请求外网时,返回200 OK，且走了父代理
```
curl -x 10.206.216.96:3128 https://www.baidu.com 
返回200 OK, 且父代理access.log有记录
```
2、请求内网(10.206.216.0/24)，返回200 OK, 且不走父代理
```
curl -x 10.206.216.96:3128 https://10.206.216.99
返回200 OK, 且父代理access.log有记录
```

3、stop父代理，再请求外网, 此时预期的结果是超时，返回503，即Squid不会与原始服务器建立连接。
```
curl -x 10.206.216.96:3128 https://www.baidu.com 
...
超时, 返回503
```

## 参考
【1】 https://www.cmdschool.org/archives/4673
【2】 https://cloud-atlas.readthedocs.io/zh-cn/latest/web/proxy/squid/squid_socks_peer.html