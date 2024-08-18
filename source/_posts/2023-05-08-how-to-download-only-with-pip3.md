---
layout: next
title: pip3只下载不安装以及离线安装的方法
date: 2023-05-08 21:30:50
categories: Python
tags: Python
---

### 只下载不安装
```py3
pip3 download kubernetes -d ${folder}
```
如不需要下载依赖包，改用如下命令
```py3
pip3 download kubernetes --no-deps on -d ${folder}
 ```
<!-- more -->
### 离线安装
0. 新增requirements.txt，内容如下：
```
psutil
supervisor
kubernetes
```
1. 下载安装包到本地
```
pip3 download -d ${folder} -r /path/to/requirements.txt
```
2. 离线安装
```
pip3 install --no-index --find-links ${folder} -r /path/to/requirements.txt
```
