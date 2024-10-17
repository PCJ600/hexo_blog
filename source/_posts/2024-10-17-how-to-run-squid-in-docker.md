---
layout: next
title: 在Docker中运行Squid
date: 2024-10-17 19:54:35
category: Squid
tag:
- Squid
- Docker
---

## 测试环境
VMware Rocky Linux 9.4

## 实现步骤
写一个Dockerfile构建Squid镜像; 再写一个启动脚本start_squid.sh，在启动脚本中配置并运行Squid。

<!-- more -->

## 编写Dockerfile
以rockylinux9.3做基础镜像，通过yum安装Squid, 拷贝squid.conf
```
FROM rockylinux:9.3

WORKDIR /
RUN yum -y install procps net-tools squid
COPY squid.conf /etc/squid/squid.conf
COPY start_squid.sh /start_squid.sh
RUN chmod +x start_squid.sh

CMD ["sh", "-c", "/start_squid.sh"]
```

squid.conf可以在容器里装一个squid，再把`/etc/squid/squid.conf`拷出来，需要额外添加如下配置，从而以squid普通用户启动，否则启动会报错
```
cache_effective_user squid
cache_effective_group squid
cache_log /var/log/squid/cache.log
access_log /var/log/squid/access.log squid
```

## 编写启动脚本start_squid.sh
```bash
#!/bin/bash

useradd -M -s /sbin/nologin squid
mkdir -p /var/log/squid
chown -R squid:squid /var/log/squid
squid

while true ; do
    sleep 60
done
```
注：
* 添加squid用户，创建squid日志目录并通过chown正确设置属主属组
* 脚本结尾通过while死循环，防止容器退出

## 构建镜像
```
docker build -f Dockerfile -t squid:1.0 .
```

## 启动容器
```
cid=$(docker run -d --privileged=true --net=host --ulimit nofile=65535:65535 squid:1.0)
docker exec -it ${cid} /bin/bash
```

可能会遇到报错: Squid启动失败 `FATAL: xcalloc: Unable to allocate 1073741816 blocks of 432 bytes!`

解决方法：容器内查看ulimit -n的值为1073741816，这个值太大了，导致Squid分配内存失败。 可以调成65535, 具体做法是给docker run添加参数 --ulimit nofile=65535:65535

具体参考: [https://github.com/langgenius/dify/issues/4371](https://github.com/langgenius/dify/issues/4371)
