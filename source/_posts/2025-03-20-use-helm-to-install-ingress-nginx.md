---
layout: next
title: Kubernetes集群安装ingress-nginx
date: 2025-03-20 21:28:56
categories: kubernetes
tags: kubernetes
---

下载yaml文件
```
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
l
```

查看镜像
```
cat deploy.yaml | grep image
        image: registry.k8s.io/ingress-nginx/controller:v1.12.0@sha256:e6b8de175acda6ca913891f0f727bca4527e797d52688cbe9fec9040d6f6b6fa
        image: registry.k8s.io/ingress-nginx/kube-webhook-certgen:v1.5.0@sha256:aaafd456bda110628b2d4ca6296f38731a3aaf0bf7581efae824a41c770a8fc4
        image: registry.k8s.io/ingress-nginx/kube-webhook-certgen:v1.5.0@sha256:aaafd456bda110628b2d4ca6296f38731a3aaf0bf7581efae824a41c770a8fc4
```
修改deploy.yaml, 把镜像替换为国内源
```
image: swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io/ingress-nginx/controller:v1.12.0
image: swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io/ingress-nginx/kube-webhook-certgen:v1.5.0
image: swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io/ingress-nginx/kube-webhook-certgen:v1.5.0
```

修改Deployment, 指定副本数为3, hostNetwork=true 
```
apiVersion: apps/v1
kind: Deployment
metadata:
  # ...
spec:
  replicas: 3 # 设置副本数
  # ...
    spec:
      hostNetwork: true	# 设置宿主机网络
      # ...
```

给每个工作节点打上标签，允许Ingress调度到工作节点
```
# kubectl get nodes
NAME          STATUS   ROLES           AGE     VERSION
k8s-master1   Ready    control-plane   5d23h   v1.28.2
k8s-master2   Ready    control-plane   5d22h   v1.28.2
k8s-master3   Ready    control-plane   5d22h   v1.28.2
k8s-slave1    Ready    <none>          5d22h   v1.28.2
k8s-slave2    Ready    <none>          5d22h   v1.28.2
k8s-slave3    Ready    <none>          5d22h   v1.28.2

kubectl label nodes k8s-slave1 ingress-ready=true kubernetes.io/os=linux
kubectl label nodes k8s-slave2 ingress-ready=true kubernetes.io/os=linux
kubectl label nodes k8s-slave3 ingress-ready=true kubernetes.io/os=linux
```

安装Ingress
```
kubectl apply -f ingress-nginx.yaml
```

查看ingress Pod启动成功
```
# kubectl -n ingress-nginx get pods
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-cz5h4        0/1     Completed   0          82s
ingress-nginx-admission-patch-pxvd5         0/1     Completed   2          82s
ingress-nginx-controller-7fd85d4496-294nk   1/1     Running     0          82s
ingress-nginx-controller-7fd85d4496-97x8w   1/1     Running     0          82s
ingress-nginx-controller-7fd85d4496-9fk2l   1/1     Running     0          82s
```

测试OK
```
curl k8s-slave1
404
curl k8s-slave2
404
curl k8s-slave3
404 
```

添加haproxy配置, 通过LB的80端口访问
```
frontend http-in
    bind *:80
    default_backend http-servers

backend http-servers
    balance roundrobin
    server node1 k8s-slave1:80 check
    server node2 k8s-slave2:80 check
    server node3 k8s-slave3:80 check
```

创建两个namespace, 部署三个服务

Service A (service-a)
Service B (service-b)
Service C (service-c)

## 制作charts
我们平时在日常生活中会经常在不同的平台上与各种各样的应用打交道，比如从苹果的 App Store 里下载的淘宝、高德、支付宝等应用，或者是在 PC 端安装的Word、Photoshop、Steam。这些各类平台上的应用程序，对用户而言，大多只需要点击安装就可使用。
然而，在云（Kubernetes）上，部署一个应用往往却不是那么简单。如果想要部署一个应用程序到云上，首先要准备好它所需要的环境，打包成 Docker 镜像。进而把镜像放在部署文件（Deployment）中、配置服务（Service）、应用所需的账户（ServiceAccount）及权限（Role）、命名空间（Namespace）、密钥信息（Secret）、可持久化存储（PersistentVolumes）等资源。也就是编写一系列互相相关的 YAML 配置文件，将它们部署在 Kubernetes 集群上。
但是即便应用的开发者可以把这些Docker镜像存放在公共仓库中，并且将部署所需的 YAML 资源文件提供给用户，用户仍然需要自己去寻找这些资源文件，并把它们一一部署。倘若用户希望修改开发者提供的默认资源，比如使用更多的副本（Replicas）或是修改服务端口（Port），他还需要自己去查应该在这些资源文件的哪些地方修改，更不用提版本变更与维护会给开发者和用户造成多少麻烦了。可见最原始的 Kubernetes 应用形态并不便利。
https://developer.aliyun.com/article/718387

TODO: 
* 利用helm制作charts包, 滚动更新, 做一个最简单的包 (TODO)
* 测试coredns (服务之前如何通信的)