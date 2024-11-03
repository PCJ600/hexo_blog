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
本人使用的是阿里云服务器，所以配了个阿里云的

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