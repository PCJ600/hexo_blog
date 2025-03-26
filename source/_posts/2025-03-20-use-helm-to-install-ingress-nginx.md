---
layout: next
title: 'Helm快速上手: 使用Helm安装nginx-ingress'
date: 2025-03-20 21:28:56
categories: Kubernetes
tags: Kubernetes
---

# 什么是Helm? Helm解决了哪些问题?
Helm是Kubernetes的包管理工具，类似于Linux中的apt或yum. Helm通过模板化和版本控制等机制, 帮助用户快速发现、共享和使用Kubernetes应用

在Helm出现之前，部署Kubernetes应用存在以下问题:
**1. YAML配置复杂，部署和管理Kubernetes应用不方便**
* 问题：需要编写大量YAML文件定义Kubernetes资源。 随着项目规模增长, 维护这些配置文件变得困难, 尤其是在多环境部署时
* Helm解决方案: 通过Helm模板功能，开发者可以创建一个Chart, 包含所有必要的Kubernetes资源定义, 并使用变量代替硬编码值

**2. 更新或回滚到特定版本较为困难**
* 问题：缺乏版本控制功能，难以更新或回滚到指定版本。如果直接修改线上 YAML 文件，可能导致问题且难以恢复到稳定状态
* Helm解决方案: Helm 通过Release概念支持版本控制，每次部署或更新应用时都会创建一个新的Release. 如果新版本出现问题, 可以通过helm rollback快速回滚到之前的稳定版本

**3. 难以共享或复用k8s配置, 不利于多团队协作**
* 问题：在一个团队内部，不同项目可能有相似需求（如都需要部署同一套系统）。然而，由于缺少有效的共享机制，每个项目需要从头开始配置，浪费了大量时间精力
* Helm解决方案: Helm Charts 可以通过仓库共享，每个人可以从仓库中下载官方或其他团队成员发布的Charts，也可以上传自己的Charts，实现了Kubernetes配置的共享和复用

# 快速上手Helm

## Helm的三个基本概念
* Chart, 一个Helm安装包，包含了运行一个应用所需要的镜像、依赖和资源定义等
* Release, 在Kubernetes集群上运行的一个Chart实例
* Repository, 发布和存储Chart的仓库

<!-- more -->

## 安装Helm
首先安装Kubernetes集群, 参考: [kubeadm+keepalived+HAproxy搭建高可用kubernetes集群](https://blog.csdn.net/pcj_888/article/details/144240636)

查看kubernetes版本
```
# kubectl version
Server Version: v1.28.2
```

根据[官方文档](https://helm.sh/docs/topics/version_skew/)找到匹配的 Helm 版本进行安装。例如，我的 Kubernetes 版本是 v1.28.2，因此选择安装 Helm v3.16.0：
```
Helm Version	Supported Kubernetes Versions
3.17.x	1.32.x - 1.29.x
3.16.x	1.31.x - 1.28.x
```
安装Helm方法如下:
```
curl -fsSL -o helm-v3.16.0-linux-amd64.tar.gz https://get.helm.sh/helm-v3.16.0-linux-amd64.tar.gz
tar -zxvf helm-v3.16.0-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
helm version
version.BuildInfo{Version:"v3.16.0", GitCommit:"0d439e1a09683f21a0ab9401eb661401f185b00b", GitTreeState:"clean", GoVersion:"go1.22.6"}
```

# 使用Helm安装Nginx Ingress

**步骤1: 添加官方Helm仓库**
```
helm repo add my-ingress https://kubernetes.github.io/ingress-nginx
```
查看已添加的仓库
```
helm repo list
NAME            URL
my-ingress      https://kubernetes.github.io/ingress-nginx
```
更新软件源
```
# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "my-ingress" chart repository
Update Complete. ⎈Happy Helming!⎈
```

**步骤2: 查找合适的Nginx Ingress版本**
```
# helm search repo ingress-nginx
NAME                            CHART VERSION   APP VERSION     DESCRIPTION
my-ingress/ingress-nginx        4.12.0          1.12.0          Ingress controller for Kubernetes using NGINX a...

[root@k8s-master1 ~]# helm search repo ingress-nginx --versions | head -n 4
NAME                            CHART VERSION   APP VERSION     DESCRIPTION
my-ingress/ingress-nginx        4.12.0          1.12.0          Ingress controller for Kubernetes using NGINX a...
my-ingress/ingress-nginx        4.11.4          1.11.4          Ingress controller for Kubernetes using NGINX a...
my-ingress/ingress-nginx        4.11.3          1.11.3          Ingress controller for Kubernetes using NGINX a...
```

查阅[ingress-nginx官方文档](https://github.com/kubernetes/ingress-nginx), 确认与Kubernetes 1.28匹配的Ingress版本范围为v1.9.0 - v1.12.0, 我选择安装v1.11.4

**步骤3: 下载Nginx Ingress, 并修改配置**
下载指定版本的Chart
```
helm pull my-ingress/ingress-nginx --version 4.11.4
```
解压并修改配置文件
```
tar xf ingress-nginx-4.11.4.tgz

# 修改配置文件values.yaml
sed -i '/registry:/s#registry.k8s.io#swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io#g' ingress-nginx/values.yaml
sed -ri '/digest:/s@^@#@' ingress-nginx/values.yaml
sed -i '/hostNetwork:/s#false#true#' ingress-nginx/values.yaml
sed -i  '/dnsPolicy/s#ClusterFirst#ClusterFirstWithHostNet#' ingress-nginx/values.yaml
sed -i '/kind/s#Deployment#DaemonSet#' ingress-nginx/values.yaml 
sed -i 's/type: LoadBalancer/type: NodePort/' ingress-nginx/values.yaml
```
说明:
* 镜像改成国内的, 国内由于网络问题无法下载海外镜像
* 注释掉digest, 因为国内镜像可能被重新构建过, digest并不相同
* hostNetwork设为true, 直接监听宿主机80和443端口 (可选)
* 如果设置了hostNetwork为true, 建议将dnsPolicy改为ClusterFirstWithHostNet, 以便使用k8s内部的service名称解析
* Deployment改成DaemonSet类型, 每个节点部署一个Pod (可选)
* 将LoadBalancer类型改为NodePort, 适用于本地测试环境

**步骤4: 创建Namespace并安装Nginx Ingress**
为ingress-nginx创建namespace
```
kubectl create namespace ingress-nginx
```

使用helm安装ingress-nginx
```
helm install ingress-nginx ./ingress-nginx --namespace ingress-nginx
```

**步骤5: 验证**
检查Pod状态
```
# kubectl -n ingress-nginx get pods  -o wide
NAME                             READY   STATUS    RESTARTS   AGE   IP               NODE         NOMINATED NODE   READINESS GATES
ingress-nginx-controller-bst66   1/1     Running   0          38s   192.168.52.101   k8s-slave1   <none>           <none>
ingress-nginx-controller-g6b2t   1/1     Running   0          38s   192.168.52.103   k8s-slave3   <none>           <none>
ingress-nginx-controller-hsm5d   1/1     Running   0          38s   192.168.52.102   k8s-slave2   <none>           <none>
```
检查Service状态
```
# kubectl -n ingress-nginx get svc
NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.98.244.186   <none>        80:30158/TCP,443:30756/TCP   48s
ingress-nginx-controller-admission   ClusterIP   10.99.235.151   <none>        443/TCP                      48s
```
访问测试
```
从集群内部访问
# curl 10.110.172.220:80
404 Not Found (404是OK的, 说明80端口已LISTEN, Nginx响应了请求) 

# 从节点访问
curl k8s-slave1:80

# 通过 NodePort 访问
curl k8s-master1:31481
```

**卸载应用**
查看当前安装的应用
```
# helm -n ingress-nginx list
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
ingress-nginx   ingress-nginx   1               2025-03-23 13:14:28.226951332 +0800 CST deployed        ingress-nginx-4.11.4    1.11.4
```
使用`helm uninstall`卸载应用
```
helm -n ingress-nginx uninstall ingress-nginx
```

# Ingress配置SSL证书
TODO

# 参考
【1】 [https://docs.daocloud.io/kpanda/user-guide/helm/](https://docs.daocloud.io/kpanda/user-guide/helm/)
【2】 [使用Helm管理kubernetes应用](https://jimmysong.io/kubernetes-handbook/practice/helm.html)
【3】 [Ingress-Nginx使用指南](https://www.cnblogs.com/yinzhengjie/p/17975829)
【4】 [Helm 快速上手](https://wiki.opskumu.com/kubernetes/helm/helm-quickstart)
