---
layout: next
title: curl代理用户名或密码出现特殊字符时需要转义
date: 2024-07-04 22:04:05
categories: curl
tags: curl
---

举例：使用代理`127.0.0.1:3128`访问百度, 用户名`peter`, 密码`123!`

密码中包含`！`特殊字符，需要转义。 查询[在线URL编码工具](https://www.urlencoder.org/zh/), `%21`是`!`的URL编码，`curl`使用方法如下：
```
curl -x peter:123%21@127.0.0.1:3128 https://www.baidu.com
```

参考
[https://www.urlencoder.org/zh/](https://www.urlencoder.org/zh/)
