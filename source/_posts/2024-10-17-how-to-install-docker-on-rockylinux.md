---
layout: next
title: 在Rocky Linux上安装Docker
date: 2024-10-17 19:56:32
category: Docker
tags: Docker
---

## 添加docker repo, 更新源, 安装docker-ce
```
dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
dnf update
dnf install -y docker-ce
```

## 启动docker
```
systemctl daemon-reload
systemctl start docker
```

<!-- more -->

## 查看docker版本
```
docker version
Client: Docker Engine - Community
 Version:           27.3.1
```

## 下载镜像
```
docker pull rockylinux:9.3
```
## 参考
[https://www.rockylinux.cn/notes/zai-rocky-linux-9-1-shang-an-zhuang-docker-ce.html](https://www.rockylinux.cn/notes/zai-rocky-linux-9-1-shang-an-zhuang-docker-ce.html)
