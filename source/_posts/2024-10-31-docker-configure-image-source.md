---
layout: next
title: Docker配置国内源加速
date: 2024-10-31 22:58:13
categories: Docker
tags: Docker
---

## 配置国内源加速
添加配置文件`/etc/docker/daemon.json`, 内容如下：
```
{
  "registry-mirrors": ["https://6kx4zyno.mirror.aliyuncs.com"]
}
```
本人使用的是阿里云服务器，所以配了个阿里云的。每个阿里云账号都可以通过以下步骤获取一个镜像加速地址
* 访问阿里云网站(https://www.aliyun.com)，登录账号
* 产品栏搜索(容器镜像服务)ACR
* 镜像工具 -> 镜像加速器 -> 获取镜像加速地址

## 使配置生效
```
systemctl daemon-reload
systemctl restart docker
```

## 测试
```
docker pull busybox
```

<!-- more -->
## 参考
https://www.cnblogs.com/data101/p/18248292