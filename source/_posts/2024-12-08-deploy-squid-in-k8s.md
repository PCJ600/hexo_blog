---
layout: next
title: 【k8s实践】 部署Squid
date: 2024-12-08 14:54:35
category:
- Kubernetes
tag:
- Kubernetes
- Squid
- Docker
---

# 前言
学习如何在k8s中部署Squid, 过程如下:
* 先在Docker环境中运行Squid, 准备好Squid镜像。
* 再搭建一个k8s环境，把上一步的docker镜像传进来，再创建对应k8s资源，完成Squid的部署

# 准备环境
准备两台Linux VM:
* 一台Centos/RHEL/RockyLinux 7/8/9，安装Docker, 构建Squid的Docker镜像
* 一台RockyLinux 9, 用于运行k8s, 部署Squid

# 1. 在Docker环境中运行Squid, 准备Squid镜像
 
## 安装Docker
参考: [安装Docker](https://blog.csdn.net/pcj_888/article/details/143018460)
<!-- more -->

## 准备Squid主配置文件squid.conf  
我是基于rocky9.3的镜像, 通过yum install方式安装Squid, 默认主配置文件路径在容器中的`/etc/squid/squid.conf`
使用默认的squid.conf, 无法在容器中正常启动Squid。 需要额外添加如下配置，从而以squid普通用户启动
```
cache_effective_user squid
cache_effective_group squid
cache_log /var/log/squid/cache.log
access_log /var/log/squid/access.log squid
```
你可以在rocky9.3的容器里装一个squid，把`/etc/squid/squid.conf`拷出来，再添加以上配置即可; 或者直接使用如下的squid.conf
```
#
# Recommended minimum configuration:
#

# Example rule allowing access from your local networks.
# Adapt to list your (internal) IP networks from where browsing
# should be allowed
acl localnet src 0.0.0.1-0.255.255.255  # RFC 1122 "this" network (LAN)
acl localnet src 10.0.0.0/8     # RFC 1918 local private network (LAN)
acl localnet src 100.64.0.0/10      # RFC 6598 shared address space (CGN)
acl localnet src 169.254.0.0/16     # RFC 3927 link-local (directly plugged) machines
acl localnet src 172.16.0.0/12      # RFC 1918 local private network (LAN)
acl localnet src 192.168.0.0/16     # RFC 1918 local private network (LAN)
acl localnet src fc00::/7           # RFC 4193 local private network range
acl localnet src fe80::/10          # RFC 4291 link-local (directly plugged) machines

acl SSL_ports port 443
acl Safe_ports port 80      # http
acl Safe_ports port 21      # ftp
acl Safe_ports port 443     # https
acl Safe_ports port 70      # gopher
acl Safe_ports port 210     # wais
acl Safe_ports port 1025-65535  # unregistered ports
acl Safe_ports port 280     # http-mgmt
acl Safe_ports port 488     # gss-http
acl Safe_ports port 591     # filemaker
acl Safe_ports port 777     # multiling http

#
# Recommended minimum Access Permission configuration:
#
# Deny requests to certain unsafe ports
http_access deny !Safe_ports

# Deny CONNECT to other than secure SSL ports
http_access deny CONNECT !SSL_ports

# Only allow cachemgr access from localhost
http_access allow localhost manager
http_access deny manager

# We strongly recommend the following be uncommented to protect innocent
# web applications running on the proxy server who think the only
# one who can access services on "localhost" is a local user
#http_access deny to_localhost

#
# INSERT YOUR OWN RULE(S) HERE TO ALLOW ACCESS FROM YOUR CLIENTS
#

# Example rule allowing access from your local networks.
# Adapt localnet in the ACL section to list your (internal) IP networks
# from where browsing should be allowed
http_access allow localnet
http_access allow localhost

# And finally deny all other access to this proxy
http_access deny all

# Squid normally listens to port 3128
http_port 3128

# Uncomment and adjust the following to add a disk cache directory.
#cache_dir ufs /var/spool/squid 100 16 256

# Leave coredumps in the first cache dir
coredump_dir /var/spool/squid

#
# Add any of your own refresh_pattern entries above these.
#
refresh_pattern ^ftp:       1440    20% 10080
refresh_pattern -i (/cgi-bin/|\?) 0 0%  0
refresh_pattern .       0   20% 4320

cache_effective_user squid
cache_effective_group squid
cache_log /var/log/squid/cache.log
access_log /var/log/squid/access.log squid
```
 
## 编写启动脚本start_squid.sh
编写一个启动脚本，作为容器的CMD。目的是在启动Squid前做一些必要的配置工作
* 添加Squid用户
* 创建Squid日志目录并正确设置属主属组，防止Squid启动报错

脚本内容如下
```bash
#!/bin/bash

useradd -M -s /sbin/nologin squid
mkdir -p /var/log/squid
chown -R squid:squid /var/log/squid
squid

# 防止容器推出
while true ; do
    sleep 60
done
```

## 编写Dockerfile
编写Dockerfile, 以rocky9.3做基础镜像，通过yum安装Squid, 用准备好的squid.conf覆盖掉默认的，再启动squid。
至此，你的项目目录中应该有如下三个文件:
```
build.sh
Dockerfile
squid.conf
```
Dockerfile内容如下:
```
FROM rockylinux:9.3

WORKDIR /
RUN yum -y install procps net-tools squid
COPY squid.conf /etc/squid/squid.conf
COPY start_squid.sh /start_squid.sh
RUN chmod +x start_squid.sh

CMD ["sh", "-c", "/start_squid.sh"]
```

## 构建并保存镜像到文件
**构建镜像**
```
docker build -f Dockerfile -t squid:1.0 .
```
**保存镜像到文件**
```
docker save -o squid_1.0.tar squid:1.0
```
`squid_1.0.tar`这个镜像文件需要传到k8s机器上, 用于后续的镜像导入

## 启动容器，测试镜像
```
cid=$(docker run -d --privileged=true --net=host --ulimit nofile=65535:65535 squid:1.0)
docker exec -it ${cid} /bin/bash

# 容器内curl测试结果为200 OK, 说明Squid代理正常工作
curl -sIL -w "%{http_code}\n" -x localhost:3128 https://www.baidu.com
HTTP/1.1 200 Connection established
...
HTTP/1.1 200 OK
```

可能会遇到这个报错: Squid启动失败 `FATAL: xcalloc: Unable to allocate 1073741816 blocks of 432 bytes!`
解决方法：
容器内执行`ulimit -n`的值为1073741816，这个值太大了，导致Squid申请了过大的内存。 这里需要给docker run添加参数`--ulimit nofile=65535:65535`

# 2. 在k8s中部署Squid
在单机k8s中部署Squid。Pod副本数设置为1, 使用宿主机网络, 监听3128端口。

## 在RockyLinux9 VM上搭建单机k8s
k8s单机环境用kubernetes, Microk8s, k3s都可以, 我用的是kubernetes; k8s搭建方式参考:
* [kubernetes搭建](https://blog.csdn.net/pcj_888/article/details/144240636)
* [Microk8s单机搭建](https://blog.csdn.net/pcj_888/article/details/144169716)


## 把Squid镜像传到k8s机器上，导入镜像
```
ctr -n k8s.io image import squid_1.0.tar
unpacking docker.io/library/squid:1.0 (sha256:3f027335651ca3d8979672e22b5202cc85dfcc22f81fe1814fa0a8dd34078cae)...done
```
如果是Microk8s环境，用如下命令导入镜像
```
microk8s.ctr image import squid_1.0.tar
```

## 创建Deployment, 设置Pod内存限制, 应用Deployment 
首先创建一个namespace，名称为squid, 和其他namespace资源隔离开。
```
kubectl create ns squid
```
创建`squid-deployment.yaml`文件, 限制Pod内存为4GiB，内容如下:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: squid
  namespace: squid # 指定namespace
  labels:
    name: squid
spec:
  replicas: 1 # 副本数为1
  selector:
    matchLabels:
      app: squid
  template:
    metadata:
      labels:
        app: squid
    spec:
      dnsPolicy: ClusterFirstWithHostNet
      hostNetwork: true # 使用宿主机网络
      containers:
        - name: squid
          image: squid:1.0 # 指定镜像名称
          imagePullPolicy: IfNotPresent # 镜像下载策略, 如果本地不存在尝试下载
          resources:
            limits:
              memory: "4Gi" # 内存限制4G
```
再应用Deployment
```
kubectl create -f squid-deployment.yaml
```

此时能观察到Pod被创建, 但是会因为内存溢出不断的重启。 这个问题的解决思路有两种:
* 设置Pod的ulimits限制，限制打开的文件数。
* 在Squid配置文件中限制最大的fd个数(max_filedescriptors)

这里采用第二种方法, 直接修改squid.conf的方法, 在配置文件最后添加一行
```
max_filedescriptors 65536
```
再重新打镜像, 导入k8s, 创建Deployment, 问题解决。 Squid Pod创建成功，正向代理功能测试OK
```
curl -x localhost:3128 https://www.baidu.com
200 OK
``` 

注: 实际应用中, 可以用squidclient测试, 为`max_filedescriptors`设置合适的值
```
squidclient mgr:info | grep 'file desc'
        Maximum number of file descriptors:   65536
        Largest file desc currently in use:     11
        Number of file desc currently in use:    6
        Available number of file descriptors: 65530
        Reserved number of file descriptors:   100
```

## 创建Volume, 把Squid日志持久化保存
实际应用中，需要把Squid的访问日志持久化保存，用于问题定位。 如果Pod遇到故障重启, 之前记录的日志也要保留下来。 在K8s中可以通过Volume实现这一点。

实现效果: 把Squid Pod中`/var/log/squid/`下的所有日志文件，持久化保存到宿主机中的/data/squidlog/
 


**创建`squid-volume.yaml`文件, 定义PV和PVC**
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: squid-volume
  namespace: squid
spec:
  capacity:
    storage: 4Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/squidlog/ # 指定宿主机路径
  claimRef:
    name: squid-claim
    namespace: squid
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: squid-claim
  namespace: squid
spec:
  volumeName: squid-volume
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 4Gi
```

**修改`squid-deployment.yaml`文件, 引用volume**
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: squid
  namespace: squid
  labels:
    name: squid
spec:
  replicas: 1
  selector:
    matchLabels:
      app: squid
  template:
    metadata:
      labels:
        app: squid
    spec:
      volumes:
        - name: squid-volume
          persistentVolumeClaim:
            claimName: squid-claim
      dnsPolicy: ClusterFirstWithHostNet
      hostNetwork: true
      containers:
        - name: squid
          image: squid:IMAGE_PLACEHOLDER
          imagePullPolicy: IfNotPresent
          resources:
            limits:
              memory: "4Gi"
          volumeMounts:
            - mountPath: /var/log/squid # 指定Pod中Squid日志路径
              name: squid-volume
```

在宿主机创建目录`mkdir -p /data/squidlog/`, 应用Volume, Deployment
```
kubectl create -f squid-volume.yaml
kubectl replace -f squid-deployment.yaml
```
确认Pod运行正常, PV和PVC创建成功
```
# kubectl -n squid get pod
NAME                    READY   STATUS    RESTARTS   AGE
squid-864dbf5fd-khhb5   1/1     Running   0          2m19s

# kubectl -n squid get pvc
N AME          STATUS   VOLUME         CAPACITY   ACCESS MODES   STORAGECLASS   AGE
squid-claim   Bound    squid-volume   4Gi        RWO                           78s

# [root@k8s-master deploy]# kubectl get pv
NAME           CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
squid-volume   4Gi        RWO            Retain           Bound    squid/squid-claim                           80s
```

测试, 先打几个HTTPS请求
```
curl -x localhost:3128 https://www.baidu.com
200 OK 
```

再查看宿主机`/data/squidlog/`目录, Squid日志已经成功挂出; 且删除Pod后日志仍然保留在宿主机
```
# ll /data/squidlog/
total 8
-rw-r-----. 1 23 23  309 Dec  8 13:46 access.log
-rw-r-----. 1 23 23 2039 Dec  8 13:46 cache.log

# kubectl -n squid delete deploy squid
deployment.apps "squid" deleted
# ll /data/squidlog/
total 8
-rw-r-----. 1 23 23  309 Dec  8 13:46 access.log
-rw-r-----. 1 23 23 2039 Dec  8 13:46 cache.log
```
