---
layout: next
title: awk 删除第一行并提取第一列
date: 2024-08-20 19:46:07
categories: awk
tags: awk
---

例：用kubectl获取某个namespace下的所有pod
```
# kubectl -n kube-system get pod
NAME                                       READY   STATUS    RESTARTS      AGE
coredns-64c6478b6c-wz58l                   1/1     Running   1 (32h ago)   15d
calico-node-xq2hw                          1/1     Running   1 (32h ago)   15d
calico-kube-controllers-6966456d6b-dkwj2   1/1     Running   1 (32h ago)   15d
metrics-server-679c5f986d-p5pcc            1/1     Running   0             32h
```

使用awk删除第一行并提取第一列，如下：
```
# kubectl -n kube-system get pod | awk 'NR > 1 {print $1}'
coredns-64c6478b6c-wz58l
calico-node-xq2hw
calico-kube-controllers-6966456d6b-dkwj2
metrics-server-679c5f986d-p5pcc
```
