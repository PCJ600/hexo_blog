---
layout: next
title: Rocky Linux上使用NVM安装Node.js 18
date: 2025-07-11 09:46:00
categories: Node.js
tags: Node.js
---

## 问题描述
Rocky Linux 9 默认 yum 安装的 Node.js 版本是16，vite启动报错：`TypeError: crypto$2.getRandomValues is not a function` ，需安装更高版本的 Node.js


## 使用nvm安装Node.js的好处
* 多版本管理，NVM 允许你安装多个不同版本的Node.js，而不需要卸载或全局替换。
* 版本隔离，每个安装的Node.js 版本都会被NVM 隔离，不会相互干扰
* 方便切换，使用 `nvm use` 命令，可以快速切换当前使用的Node.js 版本
* 避免版本冲突，通过使用NVM，可以避免全局安装Node.js 导致的潜在版本冲突问题。

<!-- more -->

## 安装步骤
### 1. 安装NVM（Node Version Manager）
```
# 安装依赖
sudo dnf install -y curl git

# 下载并安装NVM
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.5/install.sh | bash

# 加载NVM到当前shell
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
```
### 2. 安装Node.js 18
```
# 安装指定版本
nvm install 18

# 验证安装
node -v  # 显示v18.20.8
npm -v # 显示10.8.2
```

### 3. 设置为默认版本（可选）
```
nvm alias default 18
nvm use default
```

### 4. 配置环境变量（持久化）
以下内容添加到 ~/.bashrc文件末尾：
```
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # 加载NVM
```