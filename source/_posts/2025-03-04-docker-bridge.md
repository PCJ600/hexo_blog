---
layout: next
title: Docker创建自定义网桥并指定网段
date: 2025-03-04 21:48:19
top: 94
categories: Docker
tags:
- Docker
- Project
---

# 前言
docker0是Docker默认网络的核心组件, 通过虚拟网桥和NAT技术, 实现了容器间的通信以及容器与外部网络的交互。然而, docker0网段是固定的(通常是172.17.0.0/16), 为了更灵活地管理容器网络，Docker支持创建自定义网桥，允许用户指定网段。
例如, 在我以前做的一个单板仿真项目里, 每个容器用来模拟一块板, 单板的IP需要设置为172.16.0.0/16, 和docker0网段不一致。 由于这个项目部署在每个开发的工作机上, 我们决定不直接修改docker0配置, 选择了创建自定义网桥这种更灵活的方式。

# Docker网桥的工作机制
Docker在主机上创建一个虚拟网桥(docker0), 每当启动一个容器，Docker会自动创建一对虚拟网卡(veth pair), 其中一端放在容器内部作为它的网络接口, 另一端则连接到主机上的这个虚拟网桥。 通过这种方式，容器之间可以通过网桥直接通信，数据包在网桥内转发，不经过主机的物理网络接口。
如果容器访问的是外部网络, 容器发出的数据包会先通过虚拟网桥到达主机, 然后主机通过NAT将容器的私有IP替换为自己的公网IP，从而让数据包能够顺利发送到外部网络。
<!-- more -->
![](image2.png)

# 示例： 创建自定义网桥
创建自定义网桥br0, 网段为172.16.0.0/16, 创建一组容器连到网桥br0, 各容器通过eth1(172.16.0.0/16)可以互联

## 创建自定义网桥br0

创建一个新的网桥br0, 为其分配子网172.16.0.254/24
```
sudo ip link add name br0 type bridge
sudo ip link set dev br0 up
sudo ip addr add 172.16.0.254/16 dev br0
```

## 启动两个容器并连接到docker0
启动2个容器, 默认连接到docker0网桥
```
docker run -it -d --name container1 rockylinux:9.3 bash
docker run -it -d --name container2 rockylinux:9.3 bash
```

## 将容器的 eth1 连接到自定义网桥 br0
```
# 添加veth pair
ip link add veth1_a type veth peer name veth1_b
ip link set veth1_a master br0
ip link set veth1_a up

# 把veth1_b移到容器的namespace
pid_container1=$(docker inspect -f '{{.State.Pid}}' container1)
ip link set veth1_b netns $pid_container1

# veth1_b重命名为eth1
nsenter -t $pid_container1 -n ip link set veth1_b name eth1
nsenter -t $pid_container1 -n ip link set eth1 up

# 为eth1分配地址
nsenter -t $pid_container1 -n ip addr add 172.16.1.1/16 dev eth1
```
另一个容器做类似操作
```
# 添加veth pair
ip link add veth2_a type veth peer name veth2_b
ip link set veth2_a master br0
ip link set veth2_a up

pid_container2=$(docker inspect -f '{{.State.Pid}}' container2)
ip link set veth2_b netns $pid_container2

nsenter -t $pid_container2 -n ip link set veth2_b name eth1
nsenter -t $pid_container2 -n ip link set eth1 up

nsenter -t $pid_container2 -n ip addr add 172.16.1.2/16 dev eth1
```

效果:

容器A
```
# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.3  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:ac:11:00:03  txqueuelen 0  (Ethernet)
        RX packets 127  bytes 188687 (184.2 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 121  bytes 9040 (8.8 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.16.1.1  netmask 255.255.0.0  broadcast 0.0.0.0
        inet6 fe80::4a5:69ff:feb8:acc0  prefixlen 64  scopeid 0x20<link>
        ether 06:a5:69:b8:ac:c0  txqueuelen 1000  (Ethernet)
        RX packets 125  bytes 10982 (10.7 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 106  bytes 9476 (9.2 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 32  bytes 2688 (2.6 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 32  bytes 2688 (2.6 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         _gateway        0.0.0.0         UG    0      0        0 eth0
172.16.0.0      0.0.0.0         255.255.0.0     U     0      0        0 eth1
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 eth0

# ping 172.16.1.2
PING 172.16.1.2 (172.16.1.2) 56(84) bytes of data.
64 bytes from 172.16.1.2: icmp_seq=1 ttl=64 time=0.471 ms
```

容器B
```
# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.2  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:ac:11:00:02  txqueuelen 0  (Ethernet)
        RX packets 27  bytes 2006 (1.9 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 9  bytes 626 (626.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.16.1.2  netmask 255.255.0.0  broadcast 0.0.0.0
        inet6 fe80::10b0:1aff:fe2f:766d  prefixlen 64  scopeid 0x20<link>
        ether 12:b0:1a:2f:76:6d  txqueuelen 1000  (Ethernet)
        RX packets 119  bytes 10386 (10.1 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 105  bytes 9406 (9.1 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
		
# ping 172.16.1.1
PING 172.16.1.1 (172.16.1.1) 56(84) bytes of data.
64 bytes from 172.16.1.1: icmp_seq=1 ttl=64 time=0.091 ms
```

# 参考
[浅析docker容器网桥的实现原理以及docker的四种网络模式和bridge模式的具体原理](https://www.cnblogs.com/goloving/p/15133673.html)
