---
layout: next
title: Microk8s calico-kube-controller not running, Failed to initialize Calico datastore error... 解决方法
date: 2023-05-08 21:28:59
categories: k8s
tags: 
- k8s
- troubleshooting
---

## 运行环境
Rocky Linux 9
## 问题描述
microk8s calico-kube-controller pod运行失败， 报错：`Failed to initialize Calico datastore error=Get “https://10.152.183.1:443/apis/crd.projectcalico.org/v1/clusterinformations/default”: context deadline exceeded`

<!-- more -->
## 原因分析
* Rocky Linux 9 iptables版本是1.8，基于nft而不是legacy。而calico插件默认使用legacy版本，导致`cali-INPUT`链没有生成
* 如果用户在INPUT链自定义了DROP/REJECT规则, 同时`cali-INPUT`链没有生成，可能会导致k8s内部流量被丢弃，出现calico-kube-controller通过http连接apiserver超时报错.

## 解决方法
在calico-node中添加环境变量`FELIX_IPTABLESBACKEND`
```
env:
 - name: FELIX_IPTABLESBACKEND
   value: NFT
```

具体操作如下:
```bash
microk8s kubectl delete -f /var/snap/microk8s/current/args/cni-network/cni.yaml
microk8s stop
microk8s start

Edit /var/snap/microk8s/current/args/cni-network/cni.yaml
env:
 - name: FELIX_IPTABLESBACKEND
   value: NFT
microk8s kubectl apply -f /var/snap/microk8s/current/args/cni-network/cni.yaml
```
## 参考资料
[https://microk8s.io/docs/change-cidr](https://microk8s.io/docs/change-cidr)
[https://zhuanlan.zhihu.com/p/590101932](https://zhuanlan.zhihu.com/p/590101932)

