---
layout: next
title: Rocky Linux上安装Go
date: 2025-06-03 16:28:11
categories: Golang
tags: Golang
---

# 使用官方二进制包安装

## 1. 下载 Go 官方二进制包
```
cd /tmp
wget https://go.dev/dl/go1.22.3.linux-amd64.tar.gz
```
## 2. 解压并安装到 /usr/local
```
sudo rm -rf /usr/local/go   # 如果之前有旧版本先删除
sudo tar -C /usr/local -xzf go1.22.3.linux-amd64.tar.gz
```

## 3. 设置环境变量
编辑当前用户的 shell 配置文件（如 ~/.bashrc），添加如下内容
```
export PATH=$PATH:/usr/local/go/bin
export GOPATH=$HOME/go
export PATH=$PATH:$GOPATH/bin
```
保存并应用更改
```
source ~/.bashrc
```
## 4. 验证安装是否成功
```
go version
go version go1.22.3 linux/amd64
```

<!-- more -->

## 5. 测试
写一个最简单的 Go 程序
```
mkdir -p ~/go/src/hello
cd ~/go/src/hello
vi hello.go
```
hello.go
```
package main

import "fmt"

func main() {
    fmt.Println("Hello, Rocky Linux!")
}
```
运行程序
```
go run hello.go
Hello, Rocky Linux!
```
