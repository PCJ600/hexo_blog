---
layout: next
title: k8s学习笔记
date: 2024-11-27 21:37:52
categories: k8s
tags: k8s
---

# 几个概念
Infrastructure as a Service 基础设施即服务
platform as a Service 平台即服务
Software as a Service 软件即服务


# 搭建k8s集群
* minikube(学习用)
* 二进制安装
* kubeadm
* 命令行工具


搭建k8s集群 
- kubectl
- API
Pod
- 探针
- 生命周期
Label和selector
Deployment
StatefulSet
DaemonSet

HPA
Service
- ClusterIP, ExternalName, NodePort, LoadBalancer
- Ingress 

配置和存储
- 配置管理
- 持久化存储

运维管理
- Promethous
- ELK日志
- 可视化界面

目标:
* k8s各资源对象及实践
* 运用k8s各项调度策略
* 掌握k8s网络原理及应用
* 掌握pod控制器及运用
* 掌握k8s微服务DevOps实践


# k8s是什么
一个开源的，用于管理云平台中多主机上的容器化的应用。目标是让部署容器化应用简单且高效

# 为什么需要k8s
**传统部署**
war包 -> 上传服务器 -> 放到Tomcat服务器路径，重启
存在的问题: 
* 人工部署太繁琐
* 多个服务间资源竞争，没有做资源隔离

**容器化部署**
实现资源隔离: 文件系统，网络，CPU，内存，磁盘，进程隔离

# 竞品
* Apache Mesos
* Docker Swarm
* Google kubernetes (最主流)

# 集群架构和组件

## 控制面板组件
* etcd(基于键值的分布式数据库，提供了基于Raft算法实现自主的集群高可用，持久化存储)
* kube-apiserver (接口服务，基于REST风格开发k8s接口的服务)
* kube-controller-manager (控制器管理器，管理各个类型控制器，针对k8s中各种资源进行管理)
* cloud-controller-manager (云控制器管理器，第三方云平台提供的控制器)
* kube-scheduler (调度器, 负责将Pod基于一定算法，调用到更合适的节点上)

## 节点组件
* kubelet (负责POD生命周期、存储)
* kube-proxy(负责服务发现，负载均衡)
* container runtime(容器运行时环境, docker, containerd, CRI-O)

## 附加组件
* kube-dns(为整个集群提供DNS服务)
* ingress controller(外部访问服务)
* Prometheus(资源监控)
* dashboard(控制台)
* federation
* fluentd-elasticsearch(日志收集)

**Master节点**
```
                    api-server
kube-controller-manager cloud-controller-manager
                  kube-scheduler
                       etcd
```
**普通节点**
```
kubelet              kube-proxy
pod1        pod2           pod3
container-runtime 容器运行时环境
```

# 服务分类
* 无状态应用：不会对本地环境产生依赖，例如不会存储数据到本地磁盘 (例: client -> Nginx -> Web)
* 有状态应用：会对本地环境产生依赖，例如会存储数据到本地磁盘(例: client -> Web -> Redis
有状态应用的扩容，还需要考虑到数据同步，备份

# 资源的分类
* 元数据型
	* HPA (Pod自动扩容)
	* PodTemplate
	* LimitRange
* 集群型
	* Node
	* ClusterRole
	* ClusterRoleBinding
* 命名空间型
	* 无状态服务
		* ReplicationController (RC) 动态更新Pod副本数
		* ReplicaSet (RS) 动态更新Pod副本数, 通过selector选择对哪些pod生效
		* Deployment 提供更丰富的部署功能，针对RS的更高层级封装 (主流用这个, RC/RS不用)
	* 有状态服务
		* StatefulSet (网络问题，数据持久化)
			* Headless Service (对于有状态服务的DNS管理)
			* volumeClaimTemplate (用于创建持久化卷的模板)
	* 守护进程
		* DaemonSet
	* 任务/定时任务
		* Job
		* Cronjob

# 滚动升级/回滚

# Service和ingress控制器

# 其他资源
Volume, configmap(解决配置文件在容器里固定死的问题), secret, downwardAPI

<!--   实战部分  -->

# k8s安装

## Microk8s (Done)

## minikube (TODO?)
个人学习安装，不适用于生产环境
https://minikube.sigs.k8s.io/docs/start/?arch=%2Flinux%2Fx86-64%2Fstable%2Fbinary+download

## kubeadm (DOING)


# 创建Pod和Service(第一个Nginx Pod)


































https://www.zhaowenyu.com/kubernetes-doc/brief-intro/k8s-history.html
https://www.cnblogs.com/joexu01/p/16722936.html