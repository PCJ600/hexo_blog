---
layout: next
title: RPM打包时指定python自动编译版本为3.X的方法
date: 2023-02-01 21:01:35
categories: Linux
tags: 
- Linux
- RPM
---

### 指定python自动编译的版本为python3的方法
在spec文件里添加
```bash
%define  __python /usr/bin/python3
```

<!-- more -->

### RPM打包时关闭python自动编译的方法

编辑 `/usr/lib/rpm/redhat/macros`， 注释掉`brp-python-bytecompile`一行
```bash
%__os_install_post    \
    /usr/lib/rpm/redhat/brp-compress \
    %{!?__debug_package:\
    /usr/lib/rpm/redhat/brp-strip %{__strip} \
    /usr/lib/rpm/redhat/brp-strip-comment-note %{__strip} %{__objdump} \
    } \
    /usr/lib/rpm/redhat/brp-strip-static-archive %{__strip} \
    /usr/lib/rpm/brp-python-bytecompile %{__python} %{?_python_bytecompile_errors_terminate_build} \
    /usr/lib/rpm/redhat/brp-python-hardlink \
    %{!?__jar_repack:/usr/lib/rpm/redhat/brp-java-repack-jars} \
%{nil}
```
