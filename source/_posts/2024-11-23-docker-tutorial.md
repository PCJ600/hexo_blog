---
layout: next
title: Docker基础
date: 2024-11-23 19:10:48
categories: Docker
tags: Docker
---

# 什么是容器
* 容器是一种虚拟化技术，用于将应用程序和它所有依赖项打包在一起，以便在不同环境中移植和运行
* 应用或服务之间相互隔离，共享一个OS

# 什么是Docker
* 一个开源的应用容器引擎，可以让开发者打包一个应用及其依赖包到一个轻量的，可移植的容器中
* Docker将应用程序和基础设施分开，以便快速交付软件

# Docker优点
* 快速，一致地交付应用。开发，测试，运维可以直接用一套相同环境。
* 更高效的资源利用。Docker容器共享宿主机内核。
* 更轻松的迁移和扩展。Docker容器几乎可以在任意平台运行
* 更简单的更新管理。使用Dockerfile，只需小的配置修改，就能替代以往大量更新工作

<!-- more -->
# Docker和虚拟机比较
Docker作为一种轻量级的虚拟化方式，与传统的虚拟机相比具有显著优势:
* Docker容器很快，启动和停止在秒级实现
* Docker容器对系统资源需求更少，一台主机可运行数千Docker容器
* 多个容器之间共享OS

# Docker应用场景
* CI/CD (持续集成/持续部署) 

# Docker安装
[https://pcj600.github.io/2024/1017195632.html](https://pcj600.github.io/2024/1017195632.html)
# Docker配置国内源加速
[https://pcj600.github.io/2024/1031225813.html](https://pcj600.github.io/2024/1031225813.html)
# 修改Docker容器存储路径
[https://pcj600.github.io/2021/1016200721.html](https://pcj600.github.io/2021/1016200721.html)

# Docker三个核心概念
镜像，容器，仓库

## 镜像
Docker镜像是一个特殊的文件系统,提供容器运行所需程序，库，资源，配置
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

### 什么是容器
容器是镜像的一个运行实例, 通常一个容器就是一个应用或提供一个服务

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
* nsenter是一个用于进入到目标程序所在Namespace中运行命令的工具，一般用于宿主机上调试容器中运行的程序
* 典型用途是进入容器的网络命名空间。因为容器为了轻量化，通常不包含基础网络调试工具(ip,tcpdump)，调试定位很麻烦

举例：启动一个ubuntu容器，通过nsenter进入容器的命名空间
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
存放镜像的地方, 比如DockerHub，可以上传自己的镜像；主机也可以从仓库下载镜像。

# 数据卷
使用Docker过程中，需要查看容器内应用生产的数据，或者对数据备份，或者多个容器间进行数据共享，这些涉及到容器的数据管理操作
* 数据卷(Data Volumes)
* 数据卷容器(Data Volume Containers)

数据卷是一个供容器使用的特殊目录，有如下有用特性：
* 数据卷可以在容器间共享和重用
* 对数据卷修改会立刻生效
* 对数据卷的更新，不会影响镜像
* 卷会一直存在，直到没有容器使用

## 创建数据卷
**挂载一个主机目录作为数据卷**
```
docker run --rm -it -v /home/docker:/home/docker ubuntu:14.04 /bin/bash
```

## 网络配置
映射宿主机5000端口到容器的5000端口, 并查看映射端口配置
```
docker run -d -it -p 5000:5000 -p 3000:3000 ubuntu:14.04 /bin/bash
cb113dc5adb1530f3d4a259ade0d38412c87b6cdd9190f14d1a61453cbd2dd47
# docker port cb113dc5adb1530f3d4a259ade0d38412c87b6cdd9190f14d1a61453cbd2dd47
3000/tcp -> 0.0.0.0:3000
3000/tcp -> [::]:3000
5000/tcp -> 0.0.0.0:5000
5000/tcp -> [::]:5000
```

# Dockerfile创建镜像
一个基本的Dockerfile
```
FROM ubuntu

RUN apt-get install python

CMD ...
```
## FROM
格式为FROM \<image\> 或 FROM \<image\>:\<tag\>
第一条指令必须为FROM指令, 且如果在同一个Dockerfile中创建多了个镜像，可以使用多个FROM

## MAINTAINER
格式为MAINTAINER \<name\>, 指定维护者信息

## RUN
格式为RUN \<command\>，或RUN ["executable", "param1", "param2"]
前者在Shell终端中运行命令, 即`/bin/sh -c`; 后者使用exec执行，指定其他终端可用后者实现，如RUN["/bin/bash", "-c", "echo hello"]

## CMD
支持三种格式:
* CMD ["executable", "param1", "param2"]，使用exec执行，推荐方式
* CMD command param1 param2 在/bin/sh中执行, 提供给需要交互应用
* CMD ["param1,"param2"]提供给ENTRYPOINT的默认参数

指定启动容器时执行的命令, 每个Dockerfile只能有一个CMD指令, 如果指定了多个命令, 只有最后一条会被执行
如果用户启动容器时指定了运行的命令，则会覆盖掉CMD指定的命令

## EXPOSE
格式：XPOSE \<port\>
将容器中EXPOSE端口随机映射到宿主机的某个端口 (启动时需要添加-P参数)
```
from ubuntu:14.04
EXPOSE 5000
```
构建自定义镜像 `docker build -t myubuntu:1.0 -f .`
运行容器，通过-P，Docker主机自动分配一个端口映射到5000, -p可以具体指定哪个本地端口映射过来
```
docker run -d -it -P myubuntu:1.0 /bin/bash
# docker port dc2
5000/tcp -> 0.0.0.0:32768
5000/tcp -> [::]:32768
```

## ENV
格式：ENV \<key\> \<value\>
指定一个环境变量，会被后续RUN指令使用。并在容器中保持，例如:
```
ENV VERSION 9.2
RUN curl -SL http://www.xxx.com/mysql-$VERSION.tar.xz
```

## ADD
格式为ADD <src> <dest>
复制指定src到容器中的dest, src可以是Dockerfile所在目录的一个相对路径，或URL，或一个tar文件

## COPY
格式：COPY <src> <dest>
复制本地主机的src到容器中的dest, 目标路径不存在时，会自动创建
当使用本地目录为源目录时，推荐使用COPY

## ENTRYPOINT
支持两种格式：
* ENTRYPOINT ["exec", "param1", "param2"]
* ENTRYPOINT command param1 param2

ENTRYPOINT指定容器启动后执行的命令，且不可被`docker run`提供参数覆盖
每个Dockerfile只能有一个ENTRYPOINT，当指定多个ENTRYPOINT时，最后一个生效

## VOLUME
格式：VOLUME ["/data"]
创建一个可以从本地主机或其他容器挂载的挂载点

## USER
格式：USER \<user\>
指定运行容器时的用户名或UID，后续的RUN也会使用这个用户

## WORKDIR
格式：WORKDIR /path/to/workdir
为后续的RUN,CMD,ENTRYPOINT配置工作目录
可以使用多个WORKDIR指令, 后续指令如果参数是相对路径，会基于之前命令指定的路径，例如:
```
WORKDIR /a
WORKDIR b
WORKDIR c
RUN pwd
```
pwd路径:`/a/b/c`

## 创建镜像
写完Dockerfile后，可以通过docker build创建镜像
```
docker build -t <image_name>:<image_tag> -f Dockerfile .
```

# Docker核心技术

## C-S架构
* Docker daemon(`/usr/bin/dockerd`), 作为Docker服务端接收请求，默认监听`unix://var/run/docker.sock`
* Docker client(`/usr/bin/docker`), Docker客户端

Docker daemon不直接创建容器，而是请求containerd创建一个容器
containerd也不直接创建容器，而是创建一个containerd-shim进程，让这个进程操作容器
containerd-shim调用runc启动容器, runc启动完容器后会退出, containerd-shim成为容器进程的父进程
```
                    Docker Daemon
                          |
                      Containerd
                /         |         \
Containerd-shim    Containerd-shim    Containerd-shim
       |                  |                   |
      RunC               RunC                RunC 
```

## containerd
containerd是一个容器运行时，为兼容OCI标准，将运行时及其管理功能从Docker Daemon剥离
* 向上为Docker daemon提供gRPC接口, 向下通过containerd-shim结合runC
* 负责管理镜像, 容器执行
* containerd的killMode设置为process(只杀主进程), 这样升级或重启containerd时可以不杀死现有的容器

## docker-shim
每个Docker容器有一个对应的shim进程，shim进程的作用：
* 允许容器运行时(runC)再启动容器后退出，将shim作为容器父进程，这样即使containerd和dockerd都挂了, 容器依然可用
* 向containerd报告容器退出状态

## runC
* runC是Docker按照OCF(Open Container Format)标准的一种具体实现
* runC从Docker的libcontainer中迁移来，实现容器启停,资源隔离等功能

## 容器运行时接口(CRI)
CRI(Container Runtime Interface)是一个插件接口，简称CRI。使kubelet能够使用容器运行时，无需重新编译集群组件。
![](k8s-arch.png)
k8s 1.24版本弃用dockershim，直接对接Containerd
```
kubelet    <----CRI---->    CRI-containerd    <---->    containerd    ---->    container
```

## 命名空间(Namespace)
Namespace是Linux内核针对实现容器虚拟化而引入的一个强大特性
每个容器可以拥有单独命名空间，运行其中应用像是在独立操作系统中运行一样，命名空间保证容器间彼此互不影响
OS中进程共享的资源: 内核,文件系统,网络,PID,UID,IPC,内存,硬盘,CPU

### 进程命名空间
对于同一进程(同一个task_struct),在不同namespace看到的进程号不相同

### 网络命名空间
* 通过网络命名空间,实现网络隔离。
* 一个网络命名空间为进程提供了一个完全独立的网络协议栈视图，包括:网络设备接口,IPv4和IPv6协议栈,IP路由表,防火墙规则,sockets
* Docker采用虚拟网络设备(Virtual Network Device)，将不同命名空间的网络设备连接到一起。

使用brctl, 可以看到桥接到宿主机docker0网桥的虚拟网口
```
brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.0242499fb41d       no
```

### IPC命名空间
同一个IPC命名空间的进程允许进行交互，不同空间进程无法交互

### 挂载命名空间
类似chroot,将进程放到一个特性目录执行，挂载命名空间允许不同命名空间进程看到的文件结构不同，这样每个命名空间中进程所看到的文件目录彼此被隔离

### UTS命名空间
UTS命名空间允许每个容器拥有独立主机名和域名，从而虚拟出一个有独立主机名和网络空间的环境，就和网络上一台独立主机一样; 默认情况下, Docker容器主机名就是返回容器的ID

### 控制组(CGroups)
控制组是Linux内核的一个特性，用于共享资源进行隔离，限制，审计等
控制组可以提供对容器的内存，CPU，磁盘IO等资源限制和计费管理
安装Docker后,用户可以在/sys/fs/cgroup/目录下看到对Docker组应用的各种限制项
```
ls /sys/fs/cgroup/
```

## Docker容器之间怎么隔离的
Namespace机制 + Control groups技术

## 联合文件系统(UnionFS)
一种轻量级的高性能分层文件系统, 支持将文件系统中的修改信息一次提交, 并层层叠加, 同时可以将不同目录挂载到同一虚拟文件系统下
UnionFS是实现Docker镜像的技术基础，镜像通过分层进行集成。
Docker目前支持的联合文件系统: AUFS, btrfs, vfs, DeviceMapper等

**容器网络创建过程**
Docker创建一个容器时，会具体执行如下操作:
* 创建一对虚拟接口，分别放到本地主机和新容器的命名空间中
* 本地主机一端虚拟接口连接到默认docker0网桥或指定网桥上，并具有一个以veth开头的唯一名字，如veth1234
* 容器一端虚拟接口将放到新创建容器中，并修改名字作为eth0。这个接口只在容器命名空间可见
* 从网桥可用地址中获取一个空闲地址分配给容器eth0，并配置默认网关为docker0的IP地址

```

容器A(172.17.0.2)      容器B(172.17.0.3)      容器C(172.17.0.4)
      veth                 veth                   veth
          \                  |                    /
            网桥   D  O  C  K  E  R  0 (172.17.0.1)
                             |
                            NAT
                             |
                    主机网卡(192.168.52.200)
```
**容器内路由表如何配置**
默认网络模式下, 容器的默认路由设置为docker0的IP
```
宿主机
ifconfig docker0
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
		
容器内
root@89a42a95217e:/# ip addr
32: eth0@if33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
root@89a42a95217e:/# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         172.17.0.1      0.0.0.0         UG    0      0        0 eth0
172.17.0.0      *               255.255.0.0     U     0      0        0 eth0
```

Docker有4种可选网络模式bridge, host, container, none
* --net=bridge 默认值，在Docker网桥上为容器创建新的网络战
* --net=host 使用宿主机网络，拥有完全的本地主机接口访问权限, 使用--privileged=true参数，甚至允许直接配置主机网络堆栈
* --net=container 将新建容器进程放到一个已存在容器的网络栈
* --net=none 让Docker将新容器放到隔离的网络栈中，但是不进行网络配置

## Docker安全
Docker容器的安全性, 很大程度上依赖于Linux系统自身。因此评估Docker的安全性, 主要考虑以下方面:
* Linux内核的命名空间机制提供的容器隔离安全
* Linux控制组机制对容器资源的控制能力安全
* Linux内核的能力机制所带来的操作权限安全

### 内核能力机制
* 传统Unix系统对进程权限只有root和非root两种, Capability是Linux内核一个强大特性,可以提供细粒度的权限访问控制。
* Linux内核从2.2支持能力机制，将权限划分为更加细粒度的操作能力，既可作用在进程，也可以作用在文件
* 比如, 一个web服务进程绑定在一个小于1024端口，并不需要完整权限，只需授权net_bind_service能力即可
* 默认情况，Docker启动容器被严格限制只允许使用内核一部分能力

# 网络启动与配置参数

## 自定义网桥
```
# 先停止服务，删除旧网桥
systemctl stop docker
ip link set dev docker0 down
brctl delbr docker0

# 创建一个网桥docker0
brctl addbr bridge0
ip addr add 192.168.5.1/24 dev bridge0
ip link set dev bridge0 up

# 配置Docker服务，默认桥接到创建的网桥上
echo 'DOCKER_OPTS="-b=bridge0"' >> /etc/default/docker
systemctl start docker
```

### 创建一个点到点连接
```
docker run -it --rm -net=none ubuntu:14.04 /bin/bash
docker run -it --rm -net=none ubuntu:14.04 /bin/bash

mkdir -p /var/run/netns
ln -s /proc/$1/ns/net /var/run/netns/$1
ln -s /proc/$2/ns/net /var/run/netns/$2

ip link add A type veth peer name B

ip link set A setns $1
ip netns exec $1 addr add 10.1.1.1/32 dev A
ip netns exec $1 ip link set A up
ip netns exec $1 ip route add 10.1.1.2/32 dev A

ip link set B netns $2 
ip netns exec $2 addr add 10.1.1.1/32 dev B
ip netns exec $2 ip link set B up
ip netns exec $2 ip route add 10.1.1.2/32 dev B
```

## 参考
【1】《Docker技术入门与实战》—— 杨保华、戴王剑、曹亚仑编著
【2】https://docs.docker.com/get-started/docker-overview/
【3】https://cloud.tencent.com/developer/article/1868071
【4】https://cloud.tencent.com/developer/article/2327654