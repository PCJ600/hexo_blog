---
layout: next
title: 在k8s上创建第一个Pod
date: 2024-12-02 22:05:20
categories: k8s
tags: k8s
---

# 环境
* Rocky Linux9.4 x86_64 VM 
* 安装了Microk8s (参考:[Microk8s安装方法](https://pcj600.github.io/2024/1201142626.html))

说明: 其他k8s(例如: k3s, kubernetes)创建Pod的方法和Microk8s没啥区别，可以参考本文

# 目标
创建一个Nginx的Pod，映射宿主机30000端口到Pod容器的80端口；客户端能通过宿主机30000端口访问Pod容器中的Nginx服务

# 步骤
<!-- more -->

## 从国内源下载nginx:1.27.3镜像, 再导入镜像
Microk8s执行如下命令:
```
microk8s.ctr images pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/nginx:1.27.3
microk8s.ctr images tag swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/nginx:1.27.3 docker.io/library/nginx:1.27.3
```
如果是Kubernetes环境，执行如下命令: (和Microk8s大同小异)
```
ctr -n k8s.io images pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/nginx:1.27.3
ctr -n k8s.io images tag swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/nginx:1.27.3 docker.io/library/nginx:1.27.3
```

## 创建nginx的namespace
创建一个新的namespace，名称为nginx，后续在这个namespace下创建Pod
```
kubectl create ns nginx
```

## 创建并应用deployment
创建nginx-deployment.yaml文件，内容如下:
```
apiVersion: apps/v1
kind: Deployment #指定资源类型
metadata:
  name: nginx-deployment #指定deployment名称
  namespace: nginx #指定pod运行的namespace
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1 # 指定副本数(nginx pod个数)
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.27.3 #(指定镜像)
        imagePullPolicy: IfNotPresent #(指定镜像拉取策略, IfNotPresent表示如果本地有就取本地镜像，否则从网络拉取镜像
        ports:
        - containerPort: 80 # 容器内暴露的端口
```
这个yaml文件参考了官方文档： https://kubernetes.io/zh-cn/docs/tasks/run-application/run-stateless-application-deployment/
再应用这个deployment
```
kubectl apply -f nginx-deployment.yaml
deployment.apps/nginx-deployment created
```
deployment创建成功后，查看Nginx pod状态如下:
```
kubectl -n nginx get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-5c7dff4cf7-gbtsr   1/1     Running   0          2m55s
```
查看当前Nginx deployment的内容
```
kubectl -n nginx get deploy nginx-deployment -o yaml
```

## 创建并应用service, 将宿主机端口(例如30000端口)映射到Pod的80端口
创建nginx-service.yaml文件，主要字段的说明参考注释，文件内容如下:
```
apiVersion: v1
kind: Service #指定资源类型
metadata:
  name: nginx-service #指定service的名称
  namespace: nginx #指定pod运行的namespace
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
    - protocol: TCP
      targetPort: 80 # Pod容器中的端口，是Nginx程序实际监听的端口
      port: 80 # 暴露在cluster IP上的端口，提供集群内部访问service的入口, 即clusterIP:port
      nodePort: 30000 # 指定宿主机的端口, nodePort提供集群外部访问Service的能力
```
面试题: 说说看Service中port, targetPort, nodePort的作用，有什么区别?
* port: 暴露在cluster IP上的端口，提供集群内部访问service的入口，即clusterIP:port
* nodePort: 宿主机端口, 提供集群外部访问Service的能力
* targetPort: Pod内部端口，是实际应用程序监听的端口

再应用这个service
```
kubectl apply -f nginx-service.yaml
service/nginx-service created
```
执行成功后，查看service
```
kubectl -n nginx get svc
NAME            TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
nginx-service   NodePort   10.152.183.248   <none>        80:30000/TCP   2m32s
```
可以看出NodePort端口是30000，clusterIp:port为10.152.183.248:80

## 测试
通过NodePort方式可以访问Nginx
```
curl localhost:30000
...
<title>Welcome to nginx!</title>
...
```
通过clusterIp:port方式也可以访问Nginx
```
curl 10.152.183.248:80
...
<title>Welcome to nginx!</title>
...
```

# 删除Nginx Pod
删除之前创建的deployment和service资源即可，方法如下:
```
kubectl -n nginx delete deploy nginx-deployment
kubectl -n nginx delete svc nginx-service
```

## 参考
[https://kubernetes.io/zh-cn/docs/tasks/run-application/run-stateless-application-deployment/](https://kubernetes.io/zh-cn/docs/tasks/run-application/run-stateless-application-deployment/)


