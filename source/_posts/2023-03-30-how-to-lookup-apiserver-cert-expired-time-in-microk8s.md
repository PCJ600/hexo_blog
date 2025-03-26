---
layout: next
title: 如何查看Microk8s的apiserver证书过期时间
date: 2023-03-30 21:19:39
categories: Kubernetes
tags: Kubernetes
---

使用以下命令检查当前 MicroK8s 控制平面运行状态：
```
microk8s status
```
确认 MicroK8s 控制平面运行正常后，使用以下命令获取 Kubernetes 集群的配置信息：
```
microk8s config view
```
找到以下行，其中包含 API Server 的证书:
```
certificate-authority-data: ***
certificate-authority-data 给出了证书的 Base64 编码数据。
```
在终端中解码证书数据并输出详细信息：
```
openssl x509 -in <(echo "<cert-data>") -text -noout | grep "Not After"
```
将 <cert-data> 替换为第三步输出的证书编码数据，该命令将解码证书并输出其详细信息，包括过期时间, 在输出信息中找到 “Not After” 字段，它包含证书的过期日期。

microk8s证书默认路径：/var/snap/microk8s/current/certs/
