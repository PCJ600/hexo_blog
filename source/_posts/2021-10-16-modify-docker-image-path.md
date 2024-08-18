---
layout: next
title: 修改Docker容器的存储路径
date: 2021-10-16 20:07:21
categories: Docker
tags: Docker
---

## 前言
docker拉取的镜像默认存储在根路径(`/var/lib/docker`)，但根路径存储空间有限。我们可以通过挂载更大的磁盘，将docker数据迁移到挂载磁盘上, 以解决空间不足问题。具体方法如下：

<!-- more -->

## 方法

假设挂载磁盘在`/usr1`，需要将docker数据迁移到路径`/usr1`下。步骤如下：

```shell
service docker stop 				# 停止容器
mv /var/lib/docker /usr1/docker 	# 迁移docker数据到新路径，新路径只需保证在/usr1下即可
ln -s /usr1/docker /var/lib/docker  # 创建软连接
service docker start				# 重新启动docker
```

通过 `df -h`命令可确认修改生效。


