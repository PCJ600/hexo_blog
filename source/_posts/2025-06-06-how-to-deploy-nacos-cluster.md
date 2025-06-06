---
layout: next
title: Nacos 2.5.0 集群部署
date: 2025-06-06 10:22:12
tags:
---

## 环境初始化
准备3台机器: Rocky Linux 9 x86_64，IP地址如下：
* 192.168.149.220
* 192.168.149.221
* 192.168.149.222

每台机器上安装JDK 1.8及以上版本，并配置好环境变量
```
yum install -y java-1.8.0-openjdk-devel
```
配置环境变量，编辑/etc/profile文件，添加以下内容：
```
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin
```
保存后，执行source /etc/profile使配置生效。

<!-- more -->

## 安装Nacos
下载Nacos：在每台机器上分别下载Nacos 2.5.0 安装包
```
mkdir -p /usr/local/nacos
cd /usr/local/nacos
wget https://github.com/alibaba/nacos/releases/download/2.5.0/nacos-server-2.5.0.tar.gz
tar -xvf nacos-server-2.5.0.tar.gz
```
配置Nacos
在每台机器的Nacos安装目录的conf目录下，添加集群配置文件cluster.conf（如果该文件不存在，可以将cluster.conf.example复制一份并重命名为cluster.conf），内容为三台机器的IP和端口，如下：
```
192.168.149.220:8848
192.168.149.221:8848
192.168.149.222:8848
```
Nacos支持使用MySQL数据库存储数据，在每台机器的conf/application.properties文件中，修改数据库配置，例如：
```
spring.sql.init.platform=mysql
db.num=1
db.url.0=jdbc:mysql://{mysql_host}:3306/nacos?useUnicode=true&characterEncoding=utf8
db.user={mysql_user}
db.password={mysql_password}
将{mysql_host}、{mysql_user}和{mysql_password}替换为实际的MySQL数据库地址、用户名和密码  ( [安装MySQL方法](https://pcj600.github.io/2024/0916144756.html))
```
初始化nacos数据库
```
CREATE DATABASE nacos COLLATE utf8mb4_general_ci;
```
再创建nacos数据表，从Nacos的GitHub仓库获取2.5.0版本对应的数据库初始化脚本，脚本文件为mysql-schema.sql
```
USE nacos;
SOURCE /path/to/mysql-schema.sql;
```
## 启动Nacos集群
1.启动Nacos节点：在每台机器的Nacos安装目录的bin目录下，执行启动命令：
```
sh startup.sh
```
2.验证集群状态：通过浏览器访问其中任意一台Nacos节点的管理页面，例如http://192.168.149.220:8848/nacos，输入用户名和密码（默认用户名和密码均为nacos），查看集群状态，确保三个节点都正常运行。
![](image1.png)
设置Nacos开机自启动
每台机器上创建 systemd 服务文件
```
vim /etc/systemd/system/nacos.service
编辑服务文件`nacos.service`
[Unit]
Description=Nacos Server
After=network.target

[Service]
Type=forking
User=root
Group=root
ExecStart=/usr/local/nacos/nacos/bin/startup.sh
ExecStop=/usr/local/nacos/nacos/bin/shutdown.sh
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
重新加载配置，启动nacos, 设置开机自启动
systemctl daemon-reload
systemctl enable nacos
systemctl start nacos
```