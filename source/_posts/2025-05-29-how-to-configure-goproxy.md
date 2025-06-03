---
layout: next
title: Golang 配置国内代理
date: 2025-06-03 16:29:31
categories: Golang
tags: Golang
---

# 使用 GOPROXY

临时设置
```
export GOPROXY=https://goproxy.cn,direct
```
永久设置
```
go env -w GOPROXY=https://goproxy.cn,direct
```
再`go get`下载
