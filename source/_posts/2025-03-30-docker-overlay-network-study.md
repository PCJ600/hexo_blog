---
layout: next
title: 通过 Docker Swarm 集群探究 Overlay 网络跨主机通信原理
date: 2025-03-30 15:46:44
top: 102
categories: Docker
tags: Docker
---

# 什么是Overlay网络, 用于解决什么问题 ? 
Overlay网络通过在现有网络之上创建一个虚拟网络层, 解决不同主机的容器之间相互通信的问题
如果没有Overlay网络，实现跨主机的容器通信通常需要以下方法：
* 端口映射
* 使用宿主机网络模式

这些方法牺牲了容器网络隔离的优势, 并且可能导致端口冲突问题

以下搭建一个简易的Docker Swarm 集群(一主一从), 探究 Overlay 网络下不同节点上的容器间互相通信的原理

<!-- more -->

# 搭建一主一从 Docker Swarm 集群

## 0. 准备环境
准备两台Linux虚拟机(我使用Rocky Linux 9.5), 确保两台机器都安装了 Docker，并且能够通过网络互相访问。
```
192.168.52.203 swarm-master1 
192.168.52.204 swarm-slave1 
```

## 1. 初始化 Docker Swarm 集群
在主节点 swarm-master1 上初始化 Swarm 集群
```
docker swarm init --advertise-addr 192.168.52.203
```
执行成功后, 看到如下输出
```
Swarm initialized: current node (kc5nui94soxcjxi2pbqml13ig) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-3t8jt95v04knqr2tvxqd0m1j0dspw3s5hqukcwgkskikulbxgv-bw8rbewoh80240tto2os0dwtv 192.168.52.203:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```
## 2. 将从节点加入集群
复制上述输出中的 `docker swarm join` 命令，在从节点 swarm-slave1 上执行该命令：
```
docker swarm join --token SWMTKN-1-3t8jt95v04knqr2tvxqd0m1j0dspw3s5hqukcwgkskikulbxgv-bw8rbewoh80240tto2os0dwtv 192.168.52.203:2377
```
成功后, 看到如下提示
```
This node joined a swarm as a worker.
```

**验证集群状态**
```
# docker node ls
ID                            HOSTNAME                STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
kc5nui94soxcjxi2pbqml13ig *   localhost.localdomain   Ready     Active         Leader           28.0.2
suzfvrhfn2d8g5digv7u05yz4     swarm-slave1            Ready     Active                          28.0.2
```

## 3. 创建overlay网络

在主节点上创建一个 overlay 网络, 用于跨主机容器通信
```
docker network create -d overlay my-overlay-network
```
这将创建一个名为 my-overlay-network 的 overlay 网络

## 4. 每个节点部署一个Rocky Linux容器

在主节点上运行以下命令，在主节点启动一个 Rocky Linux 容器，并将其连接到 my-overlay-network 网络中
```
docker service create \
  --name rocky-master \
  --network my-overlay-network \
  rockylinux:9.3 \
  sleep infinity
```

在主节点上运行以下命令，指定容器运行在从节点上：
```
docker service create \
  --name rocky-slave \
  --constraint 'node.hostname == swarm-slave1' \
  --network my-overlay-network \
  rockylinux:9.3 \
  sleep infinity
```

检查服务的状态，确保两个容器均已启动并正常运行：
```
# docker service ls
ID             NAME           MODE         REPLICAS   IMAGE            PORTS
1cd6vg54735o   rocky-master   replicated   1/1        rockylinux:9.3
uh605ejlvsui   rocky-slave    replicated   1/1        rockylinux:9.3

# docker service ps rocky-master
ID             NAME             IMAGE            NODE            DESIRED STATE   CURRENT STATE         ERROR     PORTS
g45cjkctal9c   rocky-master.1   rockylinux:9.3   swarm-master1   Running         Running 3 hours ago

# docker service ps rocky-slave
ID             NAME            IMAGE            NODE           DESIRED STATE   CURRENT STATE         ERROR     PORTS
7kh81mby5ryd   rocky-slave.1   rockylinux:9.3   swarm-slave1   Running         Running 3 hours ago

```

## 5. 验证容器通信
进入主节点的容器，ping从节点容器, 我的从节点容器IP为 10.0.1.104
```
# ping 10.0.1.104
PING 10.0.1.104 (10.0.1.104) 56(84) bytes of data.
64 bytes from 10.0.1.104: icmp_seq=1 ttl=64 time=0.306 ms
```

如需清理 Docker Swarm 环境, 使用如下命令:

主节点停止集群
```
docker swarm leave --force
```
从节点离开集群
```
docker swarm leave --force
```
验证是否退出集群
```
docker info | grep "Swarm"
```
如果显示 Swarm: inactive，说明该节点已成功退出 Swarm。

删除服务
```
docker service rm rocky_service
```
删除Overlay网络
```
docker network rm my_overlay_network
```

# 探究 Overlay 网络原理
Docker Overlay 网络的工作流程图如下:

![](image1.png)

数据包发送流程说明:
* 容器1发出ICMP请求报文, 目标地址10.0.1.103, 根据容器1的路由表, 该报文从容器1的eth0发出
* 容器1的eth0是一个veth pair的一端, 另一端位于一个独立的network namespace中(不是宿主机的network namespace), 报文通过veth pair从容器1传送到这个独立的network namespace中的虚拟网卡(veth1)
* 在独立命名空间中, veth1连接到虚拟网桥br0, br0根据MAC地址表, 把报文转发到vxlan0设备
* vxlan0设备对原始以太网帧进行VXLAN封装, 具体过程是:
    - 加一个新的外层以太网头部, IP头部, UDP头部, 并添加VXLAN头部
    - 设置源IP为主机A的IP(192.168.52.203), 设置目标IP为主机B的IP(192.168.52.204), UDP目标端口设置为4789
    - VXLAN头部中设置VNI(VXLAN Network Identifier), 一个24位字段, 用于标识当前的VXLAN网络
* 封装后的VXLAN报文通过主机A的物理网卡ens160发送出去
* 主机B收到VXLAN报文后, 将其传递到独立命名空间中的 vxlan0 设备
* vxlan0对报文进行解封装，恢复原始的以太网帧, 解封后的数据包进入虚拟网桥br0, br0根据MAC地址表, 将报文转发到容器2的veth设备
* 最终, 报文通过veth pair到达容器2的eth0, 容器2处理该ICMP请求并做出响应


## VXLAN报文格式
![](vxlan.png)

VXLAN报文格式主要是将原始的以太网帧封装进一个UDP数据包中("MAC in UDP")，包括以下部分：
* 外层以太网头部：包含目标MAC地址、源MAC地址和以太网类型（对于VXLAN通常是0x0800）
* 外层IP头部：指定外部网络中的源IP地址和目标IP地址，用于跨网络传输
* 外层UDP头部：标准UDP头部信息，包括源端口、目的端口（通常为4789）
* VXLAN头部：包含24位的VNI（VXLAN Network Identifier），用于区分不同的租户或逻辑网络
* 原始以太网帧：被封装的数据

## 对VXLAN报文抓包
用tcpdump抓包抓ens160的VXLAN报文
```
# tcpdump -i ens160 port 4789 -nn -vv
dropped privs to tcpdump
tcpdump: listening on ens160, link-type EN10MB (Ethernet), snapshot length 262144 bytes
15:29:36.888607 IP (tos 0x0, ttl 64, id 43464, offset 0, flags [none], proto UDP (17), length 134)
    192.168.52.203.35153 > 192.168.52.204.4789: [bad udp cksum 0xeb6b -> 0x4bbc!] VXLAN, flags [I] (0x08), vni 4097
IP (tos 0x0, ttl 64, id 57583, offset 0, flags [DF], proto ICMP (1), length 84)
    10.0.1.100 > 10.0.1.103: ICMP echo request, id 20, seq 122, length 64
15:29:36.888820 IP (tos 0x0, ttl 64, id 20056, offset 0, flags [none], proto UDP (17), length 134)
    192.168.52.204.57523 > 192.168.52.203.4789: [udp sum ok] VXLAN, flags [I] (0x08), vni 4097
IP (tos 0x0, ttl 64, id 3391, offset 0, flags [none], proto ICMP (1), length 84)
    10.0.1.103 > 10.0.1.100: ICMP echo reply, id 20, seq 122, length 64
```
导出pcap文件, 再用wireshark分析 (我用的VMware NAT虚拟机, 直接在Windows物理机上用wireshark抓vmnet8的报文也可以)

![](image2.png)

![](image3.png)

# 分析过程

查看主机A的网卡和路由表
```
# ip addr
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:68:16:97 brd ff:ff:ff:ff:ff:ff
    altname enp3s0
    inet 192.168.52.203/24 brd 192.168.52.255 scope global noprefixroute ens160
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe68:1697/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether ee:70:00:b9:06:99 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
4: docker_gwbridge: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether fe:d7:43:f4:5e:0a brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global docker_gwbridge
       valid_lft forever preferred_lft forever
    inet6 fe80::fcd7:43ff:fef4:5e0a/64 scope link
       valid_lft forever preferred_lft forever
9: veth2c2d786@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker_gwbridge state UP group default
    link/ether f2:1d:db:39:8c:d7 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::f01d:dbff:fe39:8cd7/64 scope link
       valid_lft forever preferred_lft forever
392: veth04de33f@if391: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker_gwbridge state UP group default
    link/ether 5a:ac:6c:6b:55:4f brd ff:ff:ff:ff:ff:ff link-netnsid 4
    inet6 fe80::58ac:6cff:fe6b:554f/64 scope link
       valid_lft forever preferred_lft forever

# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.52.2    0.0.0.0         UG    100    0        0 ens160
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
172.18.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker_gwbridge
192.168.52.0    0.0.0.0         255.255.255.0   U     100    0        0 ens160
```

进入容器1, 查看容器1的网卡和路由表
```
# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
389: eth0@if390: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    link/ether 02:42:0a:00:01:64 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.1.100/24 brd 10.0.1.255 scope global eth0
       valid_lft forever preferred_lft forever
391: eth1@if392: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 5e:70:90:ab:28:6c brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet 172.18.0.3/16 brd 172.18.255.255 scope global eth1
       valid_lft forever preferred_lft forever

# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.18.0.1      0.0.0.0         UG    0      0        0 eth1
10.0.1.0        0.0.0.0         255.255.255.0   U     0      0        0 eth0
172.18.0.0      0.0.0.0         255.255.0.0     U     0      0        0 eth1
```
容器中ping 10.0.1.103, 查路由表, 数据包从eth0发出, 接下来查询veth pair另一端

在宿主机`/var/run/docker/netns/`下找到容器eth0另一端的veth所在的网络命名空间
```
# nsenter --net=/var/run/docker/netns/1-tic6vjs8d3 ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP mode DEFAULT group default
    link/ether 06:12:52:c7:42:88 brd ff:ff:ff:ff:ff:ff
386: vxlan0@if386: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UNKNOWN mode DEFAULT group default
    link/ether 06:12:52:c7:42:88 brd ff:ff:ff:ff:ff:ff link-netnsid 0
388: veth0@if387: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UP mode DEFAULT group default
    link/ether 1e:09:a3:80:62:d9 brd ff:ff:ff:ff:ff:ff link-netnsid 1
390: veth1@if389: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UP mode DEFAULT group default
    link/ether 9a:7c:73:4e:a6:32 brd ff:ff:ff:ff:ff:ff link-netnsid 2
```

使用nsenter进入这个namespace, 查看网卡和路由表
```
sudo nsenter --net=/var/run/docker/netns/1-tic6vjs8d3 bash
# ifconfig
br0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 10.0.1.1  netmask 255.255.255.0  broadcast 10.0.1.255
        ether 06:12:52:c7:42:88  txqueuelen 0  (Ethernet)
        RX packets 16  bytes 448 (448.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 2  bytes 108 (108.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

veth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        ether 1e:09:a3:80:62:d9  txqueuelen 0  (Ethernet)
        RX packets 64  bytes 5656 (5.5 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 74  bytes 6188 (6.0 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

veth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        ether 9a:7c:73:4e:a6:32  txqueuelen 0  (Ethernet)
        RX packets 273  bytes 25298 (24.7 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 266  bytes 25004 (24.4 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

vxlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        ether 06:12:52:c7:42:88  txqueuelen 0  (Ethernet)
        RX packets 194  bytes 16296 (15.9 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 194  bytes 16296 (15.9 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
10.0.1.0        0.0.0.0         255.255.255.0   U     0      0        0 br0
```
* 这里的veth1和容器的eth0是一对veth pair, veth1收到ICMP报文, 查路由表, 把数据包转发给br0网桥
* br0网桥根据MAC地址表, 把数据包转发给vxlan0
* vxlan0将原始以太网帧封装进一个UDP报文(端口号4789), 把目的IP替换变成主机B的IP, 源IP变成主机A的IP, 将封装后数据包发送到宿主机的物理网卡ens160

```
容器network namespace |  vxlan0所在的network namespace  |  宿主机A的network namespace  
容器eth0      ->         veth1 ->  br0  ->  vxlan0 ->     ens160
```

# 参考
【1】 [Docker跨主机Overlay网络动手实验](https://cloud.tencent.com/developer/article/2363140)
【2】 [vxlan的原理与实验](https://blog.csdn.net/u022812849/article/details/134898675)