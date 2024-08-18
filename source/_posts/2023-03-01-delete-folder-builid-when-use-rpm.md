---
layout: next
title: RPM打包删除/usr/lib/.buildid目录的方法
date: 2023-03-01 21:17:39
categories: RPM
tags: 
- Linux
- RPM
---

Add following line to spec file
```bash
%define _build_id_links none
```
参考：
[https://gohalo.me/post/linux-create-rpm-package.html](https://gohalo.me/post/linux-create-rpm-package.html)
