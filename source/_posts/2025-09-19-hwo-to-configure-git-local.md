---
layout: next
title: Git设置单个仓库用户名和邮箱的方法
date: 2025-09-19 09:51:21
categories: Git
tags: Git
---

## 前言
在多Git仓库场景下，常需为不同仓库配置不同用户信息（name/email），又不想修改全局配置。 
本文介绍在Linux环境下, 如何为当前 Git 仓库单独设置用户信息，不修改全局配置。

## 原理: Git配置三层优先级

* 仓库级配置（local）：仅作用于当前仓库，配置文件存储在仓库根目录的 .git/config 中，优先级最高。
* 用户级配置（global）：作用于当前操作系统用户的所有仓库，配置文件存储在用户目录下（Windows 为 C:\Users\用户名\.gitconfig，macOS/Linux 为 ~/.gitconfig），优先级次之。
* 系统级配置（system）：作用于当前操作系统的所有用户，配置文件存储在 Git 安装目录的 etc/gitconfig 中，优先级最低。

## 操作方法

进入目标git仓库目录, 设置仓库级用户名和邮箱

```
cd /path/to/your/git/repo
git config --local user.name "你的用户名"
git config --local user.email "你的邮箱"

# 查看配置, 确认配置生效
git config --local --list | grep user 
user.name=xxx
user.email=xxx
```

如果需要删除当前仓库配置, 执行如下:

```
git config --local --unset user.name
git config --local --unset user.email
```