---
layout: next
title: 使用wget批量下载指定类型文件
date: 2021-10-16 20:09:52
categories: wget
tags: wget
---

举例：下载所有的RPM包(文件的扩展名为rpm)

```
wget -c -r -np -k -L -p -A rpm http:XXX/
```

其中各参数意义可通过`wget -h`查看，如下：

<!-- more -->

```
wget -h
...
-c,  --continue                  resume getting a partially-downloaded file
-r,  --recursive                 specify recursive download
-np, --no-parent                 don't ascend to the parent directory
-k,  --convert-links             make links in downloaded HTML or CSS point to local files
-L,  --relative                  follow relative links only
-p,  --page-requisites           get all images, etc. needed to display HTML page
-A,  --accept=LIST               comma-separated list of accepted extensions
```


