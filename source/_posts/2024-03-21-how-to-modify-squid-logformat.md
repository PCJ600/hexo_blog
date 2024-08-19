---
layout: next
title: 如何自定义Squid日志格式, 添加请求方法等参数
date: 2024-03-21 19:57:13
categories: Squid
tags: Squid
---

查[Squid手册](http://www.squid-cache.org/Doc/config/logformat/)，找到你需要添加的参数：
```
Field name syntax keys:
	...
	%rm
	Request method
	%ru
	Request URL, without the query string
	...
```

修改Squid配置文件`/etc/squid/squid.conf`，在`logformat`开头的这一行中添加`%rm %ru`参数，如下所示：
```bash
logformat customized  %tl %ts %6tr %>a %Ss %03>Hs %>st %<st %[un %Sh %<A %mt "%{User-Agent}>h" %03<Hs "%rm %ru HTTP/%rv" %err_code
access_log  /var/log/squid/access.log customized
```
<!-- more -->

重启Squid, 查看`access.log`, 发现日志格式已成功修改
![](image1.png)
# 参考
[http://www.squid-cache.org/Doc/config/logformat/](http://www.squid-cache.org/Doc/config/logformat/)
