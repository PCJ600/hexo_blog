---
layout: next
title: pip基础用法示例
date: 2023-05-25 21:45:25
categories: Python
tags: Python
---

## 配置国内源
```
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
[install]
trusted-host = https://pypi.tuna.tsinghua.edu.cn
```

## 安装包
```
pip install XXX              # 最新版本
pip install XXX==1.0.4       # 指定版本
pip install XXX>=1.0.4       # 最低版本
```

<!-- more -->

## 卸载包
```
pip uninstall XXX
```

## 查看某个已安装包的信息
```
pip show XXX    # 查看信息
pip show -f XXX # 查看详细信息
```

## 列出所有已安装的包
```
pip list
```

## 查看本地某个wheel包的依赖
例：使用pkginfo命令查看kubernetes-26.1.0-py2.py3-none-any.whl的依赖
```
pip3 install pkginfo
/usr/local/python3/bin/pkginfo kubernetes-26.1.0-py2.py3-none-any.whl

metadata_version: 2.1
name: kubernetes
...
...
requires_python: >=3.6
requires_dist: ['certifi (>=14.05.14)', 'six (>=1.9.0)', 'python-dateutil (>=2.5.3)', 'setuptools (>=21.0.0)', 'pyyaml (>=5.4.1)', 'google-auth (>=1.0.1)', 'websocket-client (!=0.40.0,!=0.41.*,!=0.42.*,>=0.32.0)', 'requests', 'requests-oauthlib', 'urllib3 (>=1.24.2)', 'ipaddress (>=1.0.17) ; python_version=="2.7"', "adal (>=1.0.2) ; extra == 'adal'"]
provides_extras: ['adal']
```

## 只下载某个包到本地，不安装
```
pip3 download kubernetes -d ${folder}
```
如果不想下载这个包的依赖，使用--no-deps选项，用法如下：
```
pip3 download kubernetes --no-deps on -d ${folder}
```

## 离线安装，通过requirements.txt指定要安装的包
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
