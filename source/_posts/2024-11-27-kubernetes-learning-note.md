---
layout: next
title: k8s学习笔记
date: 2024-11-27 21:37:52
categories: k8s
tags: k8s
---

目标:
* k8s各资源对象及实践
* 运用k8s各项调度策略
* 掌握k8s网络原理及应用
* 掌握pod控制器及运用
* 掌握k8s微服务DevOps实践

# kubernetes简介
Google开源的容器集群管理系统，主要功能:
* 基于容器的应用部署、维护、滚动升级
* 负载均衡，服务发现
* 跨机器和跨地区的集群调度
* 自动伸缩
* 无状态服务和有状态服务
* 广泛的volume支持
* 插件机制保证扩展性

# 创建第一个k8s Pod
https://blog.csdn.net/pcj_888/article/details/144200265
TODO: 扩容缩容、滚动升级、回滚、资源限制、健康检查

# 创建k8s集群
https://blog.csdn.net/pcj_888/article/details/144240636

# kubernetes架构
一个k8s集群由分布式存储etcd, 控制节点controller，和服务节点Node组成，k8s主要组件如下：
* etcd保存整个集群状态
* apiserver提供资源操作唯一入口，提供认证、授权、访问控制、API注册和发现等机制
* controller manager负责维护集群状态，如故障检测、自动扩展、滚动更新
* kubelet负责维护容器生命周期，同时负责Volume和网络(CNI)管理
* kube-proxy负责为Service提供cluster内部的服务发现和负载均衡

![k8s-architecture.png](k8s-architecture.png)

插件Add-ons
* kube-dns负责为整个集群提供dns服务
* ingress controller为服务提供外网入口
* fluentd-elasticsearch提供集群日志采集、存储与查询
* dashboard提供GUI


# kubenetes基本概念

## Pod
Pod是一组紧密关联的容器， 是k8s调度的基本单位
Pod设计理念是支持多个容器在一个Pod中共享网络和文件系统，可通过进程间通信和文件共享的方式完成服务

## Node
Node是Pod真正运行的主机。为了管理Pod,每个Node上至少要运行container runtime、kubelet和kube-proxy

## Namespace
Namespace提供一种机制，将同一集群的资源划分为相互隔离的组
pod,service,deploymeny都是属于某一个namespace的，而node,persistentVolume不属于任何namespace

## Service
Service是应用服务的抽象，通过labels为应用提供负载均衡和服务发现。
匹配labels的Pod IP和端口列表组成endpoints, 由kube-proxy负责将服务IP负载均衡到这些endpoints上

## 声明式API
相对于命令式API, 声明式API对于重复操作的效果是稳定的，运行多次也不会出错

## Deployment
Deployment(部署)表示用户对k8s集群的一次更新操作。 可以是创建一个服务，更新一个服务，或者是滚动升级一个服务

## Service
Pod只是运行服务的实例，随时可能在一个节点停止，在另一个节点以新的IP启动，因此不能以确定IP和端口提供服务。
Service实现了服务发现和负载均衡的核心功能。
* 每个Service对应一个集群内有效的虚拟IP，集群内部通过虚拟IP访问一个服务
* k8s集群中微服务的负载均衡由kube-proxy实现。kube-proxy是一个分布式代理服务器，每个服务节点都有一个

## DaemonSet
业务Pod可能在有些节点运行多个Pod, 有些节点又没有Pod运行。而DaemonSet保证每个节点上都要有一个Pod运行。
典型的DaemonSet包括: 日志、监控(比如fluentd) 

## StatefulSet
StatefulSet是管理一组有状态pod的部署和扩展的控制器。

## Volume
k8s的volume(存储卷)和Docker的类似。Docker的volume作用范围为一个容器，k8s的volume作用范围是一个Pod
每个Pod中声明的存储卷由Pod中所有容器共享

## PV(Persistent Volume)和PVC(Persistent Volume Claim)
PV和PVC的关系，和Node与Pod关系类似

## Secret
Secret用于保存和传递密码、秘钥、认证凭证等敏感信息

## 环境变量



## ImagePullPolicy
* Always: 不管镜像是否存在都拉取
* Never: 不管镜像是否存在都不会进行拉取
* IfNotPresent：默认值，只有镜像不存在时，才会进行镜像拉取

注:
* 默认为IfNotPresent, 但:latest标签镜像默认为Always
* 生产环境要避免使用:latest标签，开发环境可借助:latest自动拉取最新镜像

## 访问DNS的策略
https://juejin.cn/post/6844903665879220231
通过设置dnsPolicy参数，设置访问DNS的策略。默认为ClusterFirst

* ClusterFirst
* ClusterFirstHostNet
* Default
* None

** ClusterFirst**
默认值, 优先使用kubedns或coredns解析，如果不成功使用宿主机DNS解析
有一个冲突，如果Pod设置了HostNetwork=true, ClusterFirst会强制转换为Default

**Default**
表示Pod里DNS配置和宿主机完全一致(/etc/resolv.conf完全一致)

**ClusterFirstWithHostNet**
Pod以host模式启动时，会使用宿主机的/etc/resolv.conf配置
如果Pod中仍然需要用k8s集群的DNS服务，需要将dnsPolicy设置为ClusterFirstWithHostNet

**None**
清除Pod预设的DNS配置。如果设置为None，为了避免Pod里没有任何DNS，需要添加dnsConfig描述自定义的DNS参数，例:
```
spec:
  dnsConfig:
    nameservers:
	  - 1.2.3.4
	searches:
	  - my.dns.search.suffix
	options:
	  - name: ndots
	    value: "2"
```

**使用主机的IPC命名空间**
设置hostIPC参数为True, 使用主机的IPC命名空间，默认为False

**使用主机的网络命名空间**
设置hostNetwork参数为True, 使用主机的网络命名空间，默认为False

**使用主机的PID空间**
设置hostPID参数为True, 使用主机的PID命名空间，默认为False


# POD几种常见的状态
* Pending 挂起
* Running Pod所有容器已创建，且至少一个容器正处于运行状态、正在启动状态或重启状态
* Succeeded Pod所有容器都执行成功后推出，并且没有处于重启的容器
* Failed Pod至少一个容器推出为失败
* Unknown kubelet故障，无法获得Pod状态

## Pod重启策略(RestartPolicy)
支持三种RestartPolicy
* Always 只要退出就重启 （默认)
* OnFailure 失败退出(exit code不为0)才重启
* Never 退出后不再重启

# 健康检查(三种探针)
https://juejin.cn/post/7163135179177852936
为了探测容器状态，k8s提供了两种探针:
* LivenessProbe 存活性探针, 如果不正常就删除容器，再根据Pod重启策略作响应动作。
* ReadinessProbe 就绪性探针，如果检测失败，将Pod的IP:Port从对应endpoint列表中删除。这种机制防止流量转发到不可用Pod上
* StartupProbe 如果应用本身启动时间过长，LivenessProbe和ReadinessProbe可能会检测失败，导致容器不停地重启。 StartupProbe探针只是在容器启动后按照配置满足一次后，不在进行后续的探测。

如果三个探针同时存在，先执行StartupProbe，禁用其他两个探针。直到满足StartupProbe，再启动其他两个探针。

## LivenessProbe和ReadinessProbe支持如下三种探测方法
* ExecAction 容器中指定的命令，退出码为0表示探测成功。
* HTTPGetAction 通过HTTP GET请求容器，如果HTTP响应码【200，400)，认为容器健康。
* TCPSocketAction 通过容器的IP地址和端口号执行TCP检查。如果建立TCP链接，则表明容器健康。

可以给探针配置可选字段，用来更精确控制Liveness和Readiness两种探针行为
* initialDelaySeconds: 容器启动后等待多少秒后探针才开始工作，默认是0秒
* periodSeconds: 执行探测的时间间隔，默认为10秒
* timeoutSeconds: 探针执行检测请求后，等待响应的超时时间，默认为1秒
* failureThreshold: 探测失败的重试次数，重试一定次数后认为失败。
* successThreshold: 探针在失败后，被视为成功的最小连续成功数。默认值是 1。 存活和启动探测的这个值必须是 1。最小值是 1。

# 容器生命周期钩子
容器生命周期钩子(Container Lifecycle Hooks)监听容器生命周期的特定事件
* postStart 容器启动后执行，这里是异步执行，无法保证一定在ENTRYPOINT之后运行。 如果失败，容器会被删除，根据RestartPolicy决定是否重启
* preStop 容器停止前执行，用于资源清理

钩子函数回调支持两种方式
* exec 容器内执行命令
* httpGet: 向指定URL发起GET请求

# 使用能力机制(Capabilities)
例如：可以给容器增加CAP_NET_ADMIN，根据需要添加或删除网卡

# 调度到指定Node
使用nodeSelector，首先给Node加标签
```
kubectl label nodes <your-node-name> disktype=ssd
```
再指定Pod只运行在带有disktype=ssd标签的Node上:
```
spec:
  nodeSelector:
    disktype: ssd
```
Page 65: 自定义hosts