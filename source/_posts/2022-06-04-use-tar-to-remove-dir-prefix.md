---
layout: next
title: tar打包时去掉目录前缀的方法
date: 2022-06-04 20:56:33
categories: Linux
tags: 
- Linux
- tar
---

## 问题描述

tar命令打包时，默认会带上文件路径，举例如下：

```shell
# tree
.
├── dir1
│   └── file1.txt
└── dir2
    ├── dir3
    │   └── file3.txt
    └── file2.txt

# tar zcf hello.tar.gz ./*
# tar tvf hello.tar.gz
drwxr-xr-x root/root         0 2022-06-03 23:20 ./dir1/
-rw-r--r-- root/root         0 2022-06-03 23:20 ./dir1/file1.txt
drwxr-xr-x root/root         0 2022-06-03 23:30 ./dir2/
-rw-r--r-- root/root         0 2022-06-03 23:20 ./dir2/file2.txt
drwxr-xr-x root/root         0 2022-06-03 23:30 ./dir2/dir3/
-rw-r--r-- root/root         0 2022-06-03 23:30 ./dir2/dir3/file3.txt
```
## 如果我打包时不想带上文件路径，怎么操作？

<!-- more -->

利用-C参数，方法如下：

```shell
# tar -zcf hello.tar.gz -C dir1/ file1.txt -C ../dir2/ file2.txt -C dir3/ file3.txt
# tar tvf hello.tar.gz
-rw-r--r-- root/root         0 2022-06-03 23:20 file1.txt
-rw-r--r-- root/root         0 2022-06-03 23:20 file2.txt
-rw-r--r-- root/root         0 2022-06-03 23:30 file3.txt
```

tar用法参考手册：

```shell
tar --help
Local file selection:
-C, --directory=DIR        change to directory DIR
```
