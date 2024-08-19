---
layout: next
title: Centos7.9上安装Docker
date: 2024-03-28 20:12:16
categories: Docker
tags: Docker
---

移除旧版本Docker
```
yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine
```

安装依赖
```
yum install -y yum-utils device-mapper-persistent-data lvm2
```
<!-- more -->
添加Docker存储库
```
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```
安装
```
yum install -y docker-ce docker-ce-cli containerd.io
```
启动Docker服务
```
systemctl start docker
```
查看Docker版本
```
docker --version
Docker version 25.0.2, build 29cf629
```
