---
layout: next
title: 如何查看Microk8s的apiserver证书过期时间
date: 2023-03-30 21:19:39
categories: k8s
tags: k8s
---

## 在线安装Microk8s
1. 安装snap
```
yum install -y epel-release
yum install -y snapd
systemctl enable --now snapd.socket
systemctl start snapd
ln -s /var/lib/snapd/snap /snap
```
<!-- more -->

2. 安装microk8s

```bash
snap install microk8s --classic --channel=1.23/stable
snap alias microk8s.kubectl kubectl
```

3. 安装microk8s插件(addons)
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
docker.io/calico/cni:v3.19.1
docker.io/calico/kube-controllers:v3.17.3
docker.io/calico/node:v3.19.1
docker.io/calico/pod2daemon-flexvol:v3.19.1
docker.io/coredns/coredns:1.8.0
k8s.gcr.io/ingress-nginx/controller:v1.5.1
k8s.gcr.io/metrics-server/metrics-server:v0.5.2
k8s.gcr.io/pause:3.1
```

## 离线安装Microk8s
1. 在可联网的环境下载Microk8s的snap安装包
```bash
snap download microk8s --channel=1.23/stable # 下载指定版本1.23
```

2. 将下载好的snap包传输到目标机器上，安装microk8s
```bash
snap ack microk8s_<版本号>.assert
snap install microk8s_<版本号>.snap --classic
```

3. 启动Microk8s服务
```bash
microk8s start
```

4. 启动microk8s插件(可选)

在可联网的环境下载Microk8s插件的Image并导出
```bash
microk8s.ctr i list | grep -vw DIGEST | grep -v ^sha256 | grep -v @sha256 | awk '{print $1}'
docker.io/calico/cni:v3.19.1
docker.io/calico/kube-controllers:v3.17.3
docker.io/calico/node:v3.19.1
docker.io/calico/pod2daemon-flexvol:v3.19.1
docker.io/coredns/coredns:1.8.0
k8s.gcr.io/ingress-nginx/controller:v1.5.1
k8s.gcr.io/metrics-server/metrics-server:v0.5.2
k8s.gcr.io/pause:3.1

microk8s.ctr i export cni.tar docker.io/calico/cni:v3.19.1
microk8s.ctr i export kube-controllers.tar docker.io/calico/kube-controllers:v3.17.3
microk8s.ctr i export .....
```
将image传到目标机器上并导入
```bash
microk8s.ctr image import *.tar
```
启动microk8s插件
```bash
microk8s enable dns ingress metrics-server
microk8s status --wait-ready
```

配置kubectl访问Microk8s
```bash
microk8s.kubectl config view --raw > $HOME/.kube/config
```

## 卸载microk8s
```bash
microk8s stop 
snap remove microk8s
rm -rf /root/.kube
```
