---
layout: next
title: 在Docker中使用GDB调试的方法
date: 2023-05-09 21:42:29
categories: GDB
tags:
- GDB
- Docker
---

## Docker中使用GDB调试的方法
1. 首先在docker中安装`gdb`， 以centos为例，可以用`yum install gdb`安装
2. 启动docker容器命令时，需要添加`--privileged`, `--cap-add=SYS_PTRACE`, `--security-opt seccomp=unconfined`参数， 如下
```bash
docker run --privileged -d -it  --cap-add=SYS_PTRACE --security-opt seccomp=unconfined [your_container_id]  bash
```
<!-- more -->
添加这些参数的原因参考： [为什么在Docker里使用gdb调试器会报错](https://developer.aliyun.com/article/674757)
