---
layout: next
title: SeaweedFS 3.97 编译方法
date: 2026-02-06 09:51:36
categories: s3
tags: s3
---

## 目标

基于SeaweedFS源码编译出 weed 二进制文件，用于后续部署

## 编译环境

Debian 12 虚拟机、安装 go1.24.1，安装基础编译工具
```bash
apt update && apt install -y git make gcc g++
```

## 编译步骤

```bash
# 1. 克隆SeaweedFS代码仓库, 切换至3.97版本
git clone git@github.com:seaweedfs/seaweedfs.git
cd seaweedfs && git checkout 3.97
# 2. 进入编译目录
cd weed/
# 3. 激活go1.24.1环境并编译安装 (使用gvm或者直接安装go1.24.1)
gvm use go1.24.1
make install
```

## 验证编译结果

```bash
echo $GOROOT
/root/.gvm/gos/go1.24.1

# 查看weed二进制文件路径
which weed
/root/.gvm/pkgsets/go1.24.1/global/bin/weed
```