---
layout: next
title: Docker教程
date: 2024-11-23 19:10:48
categories: Docker
tags: Docker
---

https://docs.docker.com/get-started/docker-overview/

# 安装Docker
[https://pcj600.github.io/2024/1017195632.html](https://pcj600.github.io/2024/1017195632.html)

# Docker简介
* 一个用于开发，发布，运行应用的开放平台，基于Go语言实现。
* Docker将应用程序和基础设施分开，以便快速交付软件。
* Docker的设计目标是"Build, Ship and Run Any App, Anywhere"

# Docker和虚拟机比较
Docker作为一种轻量级的虚拟化方式，与传统的虚拟机相比具有显著优势:
* Docker容器很快，启动和停止在秒级实现
* Docker容器对系统资源需求更少，一台主机可运行数千Docker容器
* 通过类似Git操作方便用户获取,分发,更新应用镜像
* 通过Dockerfile支持灵活的自动化创建和部署机制

# Docker优点
* 快速，一致地交付应用。开发，测试，运维可以直接用一套相同环境。
* 更高效的资源利用。Docker容器共享宿主机内核。
* 更轻松的迁移和扩展。Docker容器几乎可以在任意平台运行
* 更简单的更新管理。使用Dockerfile，只需小的配置修改，就能替代以往大量更新工作

# Docker应用场景
* CI/CD (持续集成/持续部署) 

# Docker三大核心概念
* 镜像
* 容器
* 仓库

## 镜像
Docker运行容器前需要本地存在对应的镜像。如果本地找不到镜像，Docker会尝试从镜像仓库下载

### 获取镜像
下载最新的ubuntu镜像
```
docker pull ubuntu # 或 docker pull ubuntu:latest
```
下载指定版本的ubuntu镜像，比如14.04
```
docker pull ubuntu:14.04
```
用户也可以选择从其他仓库下载镜像，此时需要指定完整的仓库地址，例如:
```
docker pull dl.dockerpool.com:5000/ubuntu
```
### 查看镜像
```
docker images
REPOSITORY   TAG       IMAGE ID       CREATED       SIZE
busybox      latest    beae173ccac6   2 years ago   1.24MB
mysql        latest    3218b38490ce   2 years ago   516MB
```

### 查看镜像详细信息
```
docker inspect [IMAGE ID]
```
docker inspect返回一个JSON格式数据。如果只需要某一项内容，使用-f参数
```
docker inspect ubuntu -f {{".Architecture"}}
amd64
```
### 删除镜像
先获取要删除镜像的IMAGE ID，再使用`docker rmi`命令
```
docker images  | grep ubuntu
ubuntu       latest    ba6acccedd29   3 years ago   72.8MB
docker rmi ba6acccedd29
```

### 存出或载入镜像
把镜像存出到本地文件，可以使用docker save命令
```
docker save -o busybox.tar busybox:latest
```
从文件busybox.tar载入镜像，使用docker load命令
```
docker load --input busybox.tar
```

### 上传镜像
一般是先对镜像打个tag，再上传
```
docker tag test:latest user_repo/test:latest
docker push user_repo/test:latest
``` 

## 容器
容器是镜像的一个运行实例

### 创建容器
```
docker create -it busybox:latest
69ed2220d4d588cdeef8d0f15ce710232512b4ef30290f8a2a41b2ae19210de8
docker ps -a
CONTAINER ID   IMAGE            COMMAND   CREATED          STATUS    PORTS     NAMES
69ed2220d4d5   busybox:latest   "sh"      10 seconds ago   Created             jovial_raman
```
docker create创建的容器处于停止状态，可以用docker start命令启动它
```
docker start 69ed2220d4d5
```

### 启动容器
docker run命令用于启动容器，它等价于先执行docker create, 再执行docker start。
以下启动一个bash中断，允许用户交互
```
docker run -it ubuntu:14.04 /bin/bash
```
交互模式下, 用户可以按Ctrl+d或输入exit命令退出容器。退出后容器处于终止状态
```
root@f21525899f99:/# ps axf
    PID TTY      STAT   TIME COMMAND
      1 pts/0    Ss     0:00 /bin/bash
     16 pts/0    R+     0:00 ps axf
root@f21525899f99:/# exit

# 退出后，容器处于终止状态
# docker ps -a
CONTAINER ID   IMAGE            COMMAND       CREATED         STATUS                     PORTS     NAMES
f21525899f99   ubuntu:14.04     "/bin/bash"   9 seconds ago   Exited (0) 6 seconds ago             amazing_bassi
```

### 守护态运行
通过-d参数，实现Docker容器以守护态(daemonized)运行
```
docker run -d -it ubuntu:14.04 /bin/bash
f53c8a9de7161fdc4b75a5a1ecdaa7dbe4f2208ea01526ec9bcf025fe6f3e874

docker exec -it f53c8a9de7161fdc4b75a5a1ecdaa7dbe4f2208ea01526ec9bcf025fe6f3e874 bash
root@f53c8a9de716:/# ps axf
    PID TTY      STAT   TIME COMMAND
     16 pts/1    Ss     0:00 bash
     31 pts/1    R+     0:00  \_ ps axf
      1 pts/0    Ss+    0:00 /bin/bash
```

### 终止容器
docker stop用于终止一个运行中的容器
```
docker stop [CONTAINER ID]
```
强行中止容器
```
docker kill [CONTAINER ID]
```
查看所有容器状态
```
docker ps -a
```

### 进入容器
docker exec用于在容器运行命令，比如进入一个容器中，启动一个bash
```
docker exec -it f53c8a9de7161fdc4b75a5a1ecdaa7dbe4f2208ea01526ec9bcf025fe6f3e874 bash
root@f53c8a9de716:/# ps axf
    PID TTY      STAT   TIME COMMAND
     16 pts/1    Ss     0:00 bash
     31 pts/1    R+     0:00  \_ ps axf
      1 pts/0    Ss+    0:00 /bin/bash
```

### nsenter
https://cloud.tencent.com/developer/article/1730699
* nsenter是一个用于进入到目标程序所在Namespace中运行命令的工具，一般用于宿主机上调试容器中运行的程序
* 典型用途是进入容器的网络命名空间。因为容器为了轻量化，通常不包含基础网络调试工具(ip,tcpdump)，调试定位很麻烦

举例:启动一个ubuntu容器，通过nsenter进入容器的命名空间
```
# 启动一个ubuntu容器
docker run -d -it ubuntu:14.04 /bin/bash
ef452d2e0c579a7f3636f7a2b6ac4740a5eff0f33cee0bbfbf876632748905e3

# 查看容器进程ID
PID=$(docker inspect --format "{{ .State.Pid }}" ef452d2e0c579a7f3636f7a2b6ac4740a5eff0f33cee0bbfbf876632748905e3)
[root@iZuf65qw76eb18m9yp8b38Z ~]# echo $PID
2931665

# 进入容器的网络命名空间
nsenter -n --target 2931665
ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
14: eth0@if15: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```
### 删除容器
```
docker rm [CONTAINER ID]
```

## 仓库
仓库(Repository)是集中存放镜像的地方
### 创建私有仓库
暂时没用到，略

## 数据卷
使用Docker过程中，需要查看容器内应用生产的数据，或者对数据备份，或者多个容器间进行数据共享，这些设计到容器的数据管理操作
* 数据卷(Data Volumes)
* 数据卷容器(Data Volume Containers)

数据卷是一个供容器使用的特殊目录，有如下有用特性：
* 数据卷可以在容器间共享和重用
* 对数据卷修改会立刻生效
* 对数据卷的更新，不会影响镜像
* 卷会一直存在，直到没有容器使用
数据卷使用，类似Linux下对目录或文件做mount操作

### 创建数据卷


## 参考
Docker技术入门与实战

## 进阶问题
cgroups
namespace
docker volume原理
