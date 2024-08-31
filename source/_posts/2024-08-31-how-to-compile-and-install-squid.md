---
layout: next
title: 源码编译并安装Squid的方法
date: 2024-08-31 13:24:17
categories: Squid
tags: Squid
---

## 问题描述
RockyLinux9.4 yum包中的Squid版本是5.5, 不是最新的版本，我需要安装最新版本的Squid。

## 源码编译并安装Squid的步骤
* 访问[Squid官网](https://www.squid-cache.org/Versions/)，查看最新的稳定版本为 6.10
* 下载6.10版本的源码，各发行版编译安装Squid的方法参考[官方文档](https://wiki.squid-cache.org/SquidFaq/CompilingSquid)

下面给出我在RockyLinux9.4 VMware虚拟机上，编译安装Squid 6.10的过程：

<!-- more -->

### 下载依赖
```bash
yum install -y perl gcc autoconf automake make sudo wget
yum install -y libxml2-devel libcap-devel libtool-ltdl-devel
```
### 下载Squid源码并编译安装
```bash
wget https://www.squid-cache.org/Versions/v6/squid-6.10.tar.gz
tar -zxvf squid-6.10.tar.gz
cd squid-6.10

./configure --prefix=/usr --includedir=/usr/include --datadir=/usr/share --bindir=/usr/sbin --libexecdir=/usr/lib/squid --localstatedir=/var --sysconfdir=/etc/squid
make -j4
make install
```

### 添加squid用户, 创建Squid日志目录并设置权限
```bash
useradd -M -s /sbin/nologin squid
mkdir -p /var/log/squid
chown -R squid:squid /var/log/squid
```

### 修改squid配置文件
修改`/etc/squid/squid.conf`，新增如下配置项：
```bash
cache_effective_user squid
cache_effective_group squid
cache_log /var/log/squid/cache.log
access_log /var/log/squid/access.log squid
```
注意事项：
* 需要指定`cache_effective_user`和`cache_effective_group`, 一般设置为squid; 注意不能设置成root，否则启动会提示错误。
* 如果不设置`cache_effective_user`和`cache_effective_group`, Squid进程的user和group为nobody，启动后会异常退出，报fopen写入cache.log没有权限(13)的错误！
* 在squid配置文件里自定义`cache_log`,`access_log`的路径为/var/log/squid。默认的路径为/var/log, squid用户没有权限写入。
### 测试Squid配置是否正确
```bash
squid -k parse
squid -z
```
如果提示报错，根据错误信息修改配置项。

### 启动Squid
```bash
squid # 启动
netstat -anpt | grep squid # Squid默认端口3128, 用netstat查看3128端口是否LISTEN
tcp6  0  0  :::3128          :::*         LISTEN    pid/(squid-1)
```
### 测试Squid
```bash
curl -x localhost:3128 https://www.baidu.com
```
### 停止Squid
```bash
squid -k shutdown
```
## 参考
[https://www.squid-cache.org/](https://www.squid-cache.org/)

