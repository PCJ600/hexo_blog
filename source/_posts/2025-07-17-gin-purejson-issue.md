---
layout: next
title: 'Gin框架输出JSON，默认会将&转义为\u0026, 如何将这个转义关闭'
date: 2025-07-17 16:13:51
categories: Golang
tags: Golang
---

## 问题描述
在Gin框架中使用c.JSON()输出JSON时，默认会将特殊字符（如&）转义为Unicode编码（如\u0026）。某些场景下会导致客户端获取的URL有问题。

## 解决方案
使用 c.PureJSON()替代 c.JSON()
```
func GetPresignedURL(c *gin.Context) {
    resp := gin.H{
        "presignedUrl": "https://example.com?param=1&key=2", // 含&的URL
    }
    c.PureJSON(http.StatusOK, resp) // 关闭转义
}
```
