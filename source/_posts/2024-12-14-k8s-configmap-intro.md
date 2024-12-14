---
layout: next
title: k8s的ConfigMap是什么, 为什么设计ConfigMap, 如何使用ConfigMap
date: 2024-12-14 16:11:18
categories: k8s
tags:
- k8s
- Squid
--- 

## ConfigMap简介, 为什么设计ConfigMap
在k8s中, ConfigMap是一种API对象, 用于将非机密的配置数据存储到键值对中。 
Configmap作用是, 把配置数据从应用代码中分隔开, 让镜像和配置文件解耦，实现了镜像的可移植性。

举个例子:
我有一个Squid(正向代理)的Pod, 需要获取用户配置的白名单做访问控制。 每个用户设置的白名单都不一样, 而且用户可以随时对白名单做增、删、改，所以这个白名单的配置不能写死在代码里。
我们可以把白名单配置存储到k8s的ConfigMap, 这样配置数据和镜像就实现了解耦，Pod中可以动态地获取白名单的配置。 

## 如何使用ConfigMap
使用ConfigMap时, Pod可以将其用作环境变量、命令行参数或存储卷中的配置文件。 
下面给出一个具体的案例，将ConfigMap用作存储卷中的配置文件，Pod中通过读取配置文件的内容，就获取到了配置信息。
<!-- more -->

## ConfigMap示例
需求描述: 我有一个Squid的Pod, 用户可以修改一些配置信息，例如：白名单, 父级代理。 要求创建一个ConfigMap存储这些配置，并且将ConfigMap用作存储卷中的配置文件.
配置信息如下:
```
whitelist: www.baidu.com:443,www.google.com:443 # 白名单
customerProxy: 192.168.52.204:3128 # Squid的父级代理
```

**步骤**
首先创建一个Squid Pod, 可以参考[Squid Pod部署](https://pcj600.github.io/2024/1208145435.html)

再创建ConfigMap, 新建文件`squid-configmap.yaml`, 内容如下:
```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: squid-configmap
  namespace: squid
data:
  whitelist: "www.baidu.com,www.google.com"
  parentProxy: "192.168.52.204:3128"
```

修改Deployment, 将ConfigMap映射到Pod的/etc/squid-config目录下。 具体做法是编辑`squid-deployment.yaml`, 添加如下内容:
```yaml
volumes:
  - name: squid-config-volume
    configMap:
      name: squid-configmap
containers:
  - name: squid
    volumeMounts:
      - mountPath: /etc/squid-config
        name: squid-config-volume
```
完整的`squid-deployment.yaml`如下
```yaml
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
        - name: squid-config-volume
          configMap:
            name: squid-configmap
      dnsPolicy: ClusterFirstWithHostNet
      hostNetwork: true
      containers:
        - name: squid
          image: squid:IMAGE_PLACEHOLDER
          imagePullPolicy: IfNotPresent
          livenessProbe:
            httpGet:
              path: /healthz
              port: 5000
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 3
          resources:
            limits:
              memory: "4Gi"
          volumeMounts:
            - mountPath: /var/log/squid
              name: squid-volume
            - mountPath: /etc/squid-config
              name: squid-config-volume
```

应用configmap, deployment, 等待Pod Ready 
```
kubectl create -f squid-configmap.yaml
kubectl -n squid delete deploy squid
kubectl create -f squid-deployment.yaml
```

**测试**
先确认ConfigMap内容包含了配置信息
```
# kubectl -n squid get cm squid-configmap -o yaml
apiVersion: v1
data:
  customerProxy: 192.168.52.204:3128
  whitelist: www.baidu.com,www.google.com
kind: ConfigMap
metadata:
  ...
```

再进入Pod，确认`/etc/squid-config`目录下的文件内容 
```
# kubectl -n squid exec -it squid-64bbc7d8f5-dqklp -- /bin/bash
cat /etc/squid-config/customerProxy
192.168.52.204:3128
# cat /etc/squid-config/whitelist
www.baidu.com,www.google.com
```

编辑configmap中的配置，等待一段时间后进入容器确认`/etc/squid-config/`的文件内容也随之更新
```
# kubectl -n squid edit cm squid-congfimap
把whitelist的值设置为www.4399.com, 等待一段时间后进入容器中，查看`/etc/squid-config/whitelist`的内容更新为www.4399.com
```

## 参考
【1】 [Kubernetes Documentation - ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
【2】 [kubernetes核心技术-ConfigMap](https://juejin.cn/post/7416195017084125236)