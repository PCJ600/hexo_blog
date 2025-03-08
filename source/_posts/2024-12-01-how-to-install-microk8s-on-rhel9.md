---
layout: next
title: 如何在RockyLinux9.4上安装Microk8s
date: 2024-12-01 14:26:26
categories: kubernetes
tags: kubernetes
---

## 在线安装Microk8s
1、安装snap
```
yum install -y epel-release
yum install -y snapd
systemctl enable --now snapd.socket
systemctl start snapd
ln -s /var/lib/snapd/snap /snap
```

2、安装microk8s
```bash
snap install microk8s --classic # 在RockyLinux9.4上, 默认下载Microk8s 1.31版本
snap install microk8s --classic --channel=1.31/stable # --channnel参数可以指定Microk8s的版本
snap alias microk8s.kubectl kubectl
```
<!-- more -->

3、安装microk8s插件(addons)
```bash
microk8s enable dns ingress metrics-server
```

成功安装后，通过kubectl查看Pod和Image
```bash
# kubectl get pods -A
kube-system   calico-node-srvfr                         1/1     Running   0          11m
kube-system   calico-kube-controllers-5b577d865-xwqjh   1/1     Running   0          11m
kube-system   coredns-64c6478b6c-fcvkk                  1/1     Running   0          23s
ingress       nginx-ingress-microk8s-controller-rqtj2   1/1     Running   0          32s
kube-system   metrics-server-679c5f986d-vb6z7           1/1     Running   0          53m

microk8s.ctr i list | grep -vw DIGEST | grep -v ^sha256 | grep -v @sha256 | awk '{print $1}'
coredns/coredns:1.10.1
docker.io/calico/cni:v3.25.1
docker.io/calico/kube-controllers:v3.25.1
docker.io/calico/node:v3.25.1
docker.io/coredns/coredns:1.10.1
registry.k8s.io/ingress-nginx/controller:v1.11.2
registry.k8s.io/metrics-server/metrics-server:v0.6.3
registry.k8s.io/pause:3.7
```

## 在线安装可能遇到的问题
**问题1:** pod启动失败, 通过`kubectl -n kube-system describe pod` 查到失败原因是下载镜像失败
解决方法: 使用microk8s.ctr从国内源下载镜像，再给镜像打个tag，需要下载的镜像如下：
```
k8s.gcr.io/pause:3.7
docker.io/calico/cni:v3.25.1
docker.io/calico/node:v3.25.1
docker.io/calico/kube-controllers:v3.25.1
docker.io/coredns/coredns:1.10.1
registry.k8s.io/ingress-nginx/controller:v1.11.2
registry.k8s.io/metrics-server/metrics-server:v0.6.3
```
从国内源下载镜像，再给镜像打tag
```
microk8s.ctr images pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io/pause:3.7
microk8s.ctr images tag  swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io/pause:3.7  registry.k8s.io/pause:3.7

microk8s.ctr images pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/calico/cni:v3.25.1
microk8s.ctr images tag  swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/calico/cni:v3.25.1  docker.io/calico/cni:v3.25.1

microk8s.ctr images pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/calico/node:v3.25.1
microk8s.ctr images tag  swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/calico/node:v3.25.1  docker.io/calico/node:v3.25.1

microk8s.ctr images pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/calico/kube-controllers:v3.25.1
microk8s.ctr images tag  swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/calico/kube-controllers:v3.25.1  docker.io/calico/kube-controllers:v3.25.1

microk8s.ctr images pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io/coredns/coredns:v1.10.1
microk8s.ctr images tag  swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io/coredns/coredns:v1.10.1  docker.io/coredns/coredns:1.10.1

microk8s.ctr images pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io/ingress-nginx/controller:v1.11.2
microk8s.ctr images tag  swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io/ingress-nginx/controller:v1.11.2  registry.k8s.io/ingress-nginx/controller:v1.11.2

microk8s.ctr images pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io/metrics-server/metrics-server:v0.6.3
microk8s.ctr images tag  swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io/metrics-server/metrics-server:v0.6.3  registry.k8s.io/metrics-server/metrics-server:v0.6.3
```

**问题2**: metrics-server启动失败, 'dial tcp XXX:10250: connect: no route to host"
解决方法：关闭防火墙，禁用SELinux
```
systemctl disable firewalld --now
setenforce 0
vim /etc/selinux/config
SELINX=enforcing这行改成SELINUX=permissive
```

开启ip_forward
```
/etc/sysctl.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
```

## 离线安装Microk8s
1、在可联网的环境下载Microk8s的snap安装包
```bash
snap download microk8s --channel=1.31/stable # 下载指定版本1.31
```

2、将下载好的snap包传输到目标机器上，安装microk8s
```bash
snap ack microk8s_<版本号>.assert
snap install microk8s_<版本号>.snap --classic
```

3、启动Microk8s服务
```bash
microk8s start
```

4、在可联网的环境下载Microk8s插件的Image并导出
```bash
microk8s.ctr i list | grep -vw DIGEST | grep -v ^sha256 | grep -v @sha256 | awk '{print $1}'
k8s.gcr.io/pause:3.7
docker.io/calico/cni:v3.25.1
docker.io/calico/node:v3.25.1
docker.io/calico/kube-controllers:v3.25.1
docker.io/coredns/coredns:1.10.1
registry.k8s.io/ingress-nginx/controller:v1.11.2
registry.k8s.io/metrics-server/metrics-server:v0.6.3

microk8s.ctr i export cni.tar docker.io/calico/cni:v3.25.1
microk8s.ctr i export kube-controllers.tar docker.io/calico/kube-controllers:v3.25.1
microk8s.ctr i export .....
```

5、将image传到目标机器上并导入
```bash
microk8s.ctr image import *.tar
```

6、启动microk8s插件
```bash
microk8s enable dns ingress metrics-server
microk8s status --wait-ready
```

7、配置kubectl访问Microk8s
```bash
microk8s.kubectl config view --raw > $HOME/.kube/config
```

## 卸载microk8s
```bash
microk8s stop 
snap remove microk8s
rm -rf /root/.kube
```
