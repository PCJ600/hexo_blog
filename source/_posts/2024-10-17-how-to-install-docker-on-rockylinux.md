---
layout: next
title: Docker安装教程
date: 2024-10-17 19:56:32
category: Docker
tags: Docker
---

# 在Rocky Linux 9上安装Docker

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

# 在Centos7上安装Docker

## 移除旧版本Docker
```
yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine
```

## 安装依赖
```
yum install -y yum-utils device-mapper-persistent-data lvm2
```

## 添加Docker存储库
```
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```
## 安装Docker
```
yum install -y docker-ce docker-ce-cli containerd.io
```

## 启动Docker服务
```
systemctl start docker
```

## 查看Docker版本
```
docker --version
Docker version 25.0.2, build 29cf629
```

## 参考
[https://www.rockylinux.cn/notes/zai-rocky-linux-9-1-shang-an-zhuang-docker-ce.html](https://www.rockylinux.cn/notes/zai-rocky-linux-9-1-shang-an-zhuang-docker-ce.html)
