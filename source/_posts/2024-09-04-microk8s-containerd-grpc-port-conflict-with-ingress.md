---
layout: next
title: Microk8s ingress启动失败, 10254端口被占用问题定位
date: 2024-09-04 20:31:11
categories: troubleshooting
tags:
- troubleshooting
- Kubernetes
---

## 问题描述
RHEL9 VM里安装了Microk8s，且使用了Nginx ingress Controller插件，443端口正常。 VM重启一次后，发现443端口没有LISTEN，不能对外提供服务。 

## 定位过程
查看ingress pod状态，为CrashLoopBackOff
```bash
# kubectl -n ingress get pods
NAME                                      READY   STATUS             RESTARTS         AGE
nginx-ingress-microk8s-controller-b6krf   0/1     CrashLoopBackOff   1102 (55s ago)   8d
```
<!-- more -->

再查看启动日志，通过`kubectl logs`命令
```bash
kubectl -n ingress logs nginx-ingress-microk8s-controller-b6krf | head -n 20
-------------------------------------------------------------------------------
NGINX Ingress controller
  Release:       v1.2.0
  Build:         a2514768cd282c41f39ab06bda17efefc4bd233a
  Repository:    https://github.com/kubernetes/ingress-nginx
  nginx version: nginx/1.19.10

-------------------------------------------------------------------------------

W0903 07:03:51.041545       7 client_config.go:617] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
I0903 07:03:51.041798       7 main.go:230] "Creating API client" host="https://10.152.183.1:443"
I0903 07:03:51.079803       7 main.go:274] "Running in Kubernetes cluster" major="1" minor="23+" git="v1.23.10-2+b9088462d1df8c" state="clean" commit="b9088462d1df8ccd2a1856d329af381fa2bce5a3" platform="linux/amd64"
I0903 07:03:51.203652       7 main.go:104] "SSL fake certificate created" file="/etc/ingress-controller/ssl/default-fake-certificate.pem"
I0903 07:03:51.259671       7 nginx.go:256] "Starting NGINX Ingress controller"
F0903 07:03:51.259727       7 main.go:345] listen tcp :10254: bind: address already in use
goroutine 117 [running]:
k8s.io/klog/v2.stacks(0x1)
        k8s.io/klog/v2@v2.60.1/klog.go:860 +0x8a
```

发现报错：`listen tcp :10254: bind: address already in use`, 10254端口是ingress用于健康检查的端口
```bash
kubectl -n ingress get ds nginx-ingress-microk8s-controller -o yaml
...
livenessProbe:
	httpGet:
		path: /healthz
		port: 10254
```

netstat查下10254端口被哪个进程占用了
```bash
[root@sg3 svc]# netstat -antp | grep containerd
tcp        0      0 127.0.0.1:1338          0.0.0.0:*               LISTEN      3631/containerd
tcp        0      0 127.0.0.1:10254         0.0.0.0:*               LISTEN      3631/containerd
```
发现是containerd占用了10254端口。因为重启前ingress是正常的，于是猜测这个containerd端口是随机分配的，接着验证一下猜测是否正确。

下一份containerd代码看看，先查containerd版本：
```
snap list
microk8s  v1.23.10  3699   -         canonical✓  classic

microk8s ctr version
Client:
  Version:  v1.5.13
```
containerd版本为v.1.5.13，找到源码 [https://github.com/containerd/containerd/releases/tag/v1.5.13](https://github.com/containerd/containerd/releases/tag/v1.5.13)

简单扫下源码，找到containerd配置文件的路径：`/var/snap/microk8s/current/args/containerd-template.toml`，内容如下：
```
[grpc]
# ......

[metrics]
	address="127.0.0.1:1338"
# ......

[ plugins. "io.containerd.grpc.v1.cri" ]
  stream_server_address = "127.0.0.1"
  stream_server_port = "0"
```

`journalctl -xeu snap.microk8s.daemon-containerd.service`查下启动日志，通过启动日志中的关键字快速定位到相关代码：
![](image1.png)
分析配置文件和代码，找到原因：**containerd配置文件中的stream_server_port默认为0, 说明监听了随机端口号，所以可能存在端口冲突。**

## 解决方法
可以给containerd指定一个端口号，防止端口冲突。 比如改成10300（具体改哪个端口根据你的情况定，这里只举个例子)，如下：
```bash
sed -i 's/stream_server_port = "[^"]*"/stream_server_port = "10300"/' /var/snap/microk8s/current/args/containerd-template.toml
microk8s stop
microk8s start
```
github上找到类似了issue：[https://github.com/containerd/containerd/issues/7097](https://github.com/containerd/containerd/issues/7097)
## 参考
[https://www.thebyte.com.cn/container/CRI-in-Kubernetes.html](https://www.thebyte.com.cn/container/CRI-in-Kubernetes.html)
