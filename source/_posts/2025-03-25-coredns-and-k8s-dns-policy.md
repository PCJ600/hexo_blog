---
layout: next
title: 理解Kubernetes中CoreDNS域名解析与DNS策略
date: 2025-03-25 23:06:50
categories: Kubernetes
tags: Kubernetes
---

# CoreDNS是什么
CoreDNS是一个灵活可扩展的DNS服务器，使用Go语言编写，旨在提供快速、灵活的DNS服务

# 为什么需要CoreDNS
CoreDNS为Kubernetes集群内部的DNS解析提供服务，使得服务之间能够通过域名互相通信
Kubernetes集群中, CoreDNS是运行在kube-system这个namespace下的Pod
```
kubectl -n kube-system get pod coredns-66f779496c-b7mmz
NAME                       READY   STATUS    RESTARTS      AGE
coredns-66f779496c-b7mmz   1/1     Running   4 (28m ago)   4d23h
```

# k8s集群中的域名是如何解析的
比如服务a访问服务b:
* 如果a和b在同一个namespace下, 可以直接在pod a中, 通过`curl b`来访问b
* 如果a和b不在同一个namespace下, 在pod a中需要通过`curl b.namespaceb`来访问b

以下动手测试
<!-- more -->

## 测试同一个namespace下的服务间域名解析
创建一个名为foo的namespace, 再创建两个Flask服务(svca, svcb)
```
kubectl get all -n foo
NAME                        READY   STATUS    RESTARTS   AGE
pod/svca-78f6c85d4-sd97h    1/1     Running   0          43s
pod/svcb-5fccb7d86b-mkqg7   1/1     Running   0          43s

NAME           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/svca   ClusterIP   10.98.115.242    <none>        8000/TCP   24s
service/svcb   ClusterIP   10.111.107.194   <none>        8000/TCP   23s

NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/svca   1/1     1            1           24s
deployment.apps/svcb   1/1     1            1           23s
```

进入pod a, 通过`curl http://svcb` 访问 b
```
kubectl exec -it pod/svca-78f6c85d4-sd97h -n foo -- sh

# curl http://svcb:8000
hello foo
```

## 测试不同namespace下的服务间域名解析
两个服务(svca, svcb), svca在foo命名空间, svcb在bar命名空间, 在pod a中访问b, 使用`curl http://svcb.bar`
```
kubectl exec -it pod/svca-78f7c85d4-sd97h -n foo -- sh
# curl http://svcb.bar:8000
hello foo
```

## 为什么同一Namespace下, 直接访问服务名`<service-name>`就可以, 不同Namespace下, 必须带上namespace(`<service-name>.<namespace>`) ?
进入pod a, 查看`/etc/resolve.conf`
```
search foo.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

这里的DNS Server(10.96.0.10)是kube-dns的ClusterIP
```
# kubectl -n kube-system get svc
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   9d
```

查看`/etc/resolv.conf`, 其中的关键点在于search指令和ndots选项：
* search：这个指令定义了一系列的后缀，当DNS查询失败时，会依次尝试将这些后缀附加到原始查询域名后面，直到找到匹配项或所有后缀都已尝试过。
* ndots：如果一个域名中包含的点（.）数量小于ndots值，则该域名会被认为是“不完整”的域名。在这种情况下，DNS 客户端会尝试将 /etc/resolv.conf 中的 search 路径逐一追加到该域名后进行解析。如果域名中的点数大于或等于 ndots，则直接将其作为完整的域名进行查询。

k8s集群中某个服务完整的域名格式是`<service-name>.<namespace>.svc.<cluster-domain>`, 验证一下:
```
# nslookup svcb.foo.svc.cluster.local 10.96.0.10
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   svcb.foo.svc.cluster.local
Address: 10.111.107.194
```

**同一命名空间下的服务访问**
CoreDNS会按照`svcb.foo.svc.cluster.local` -> `svcb.svc.cluster.local` -> `svcb.cluster.local`顺序解析, 可以看出第一次解析(`svcb.foo.svc.cluster.local`)就会成功

**不同命名空间下的服务访问**
必须使用`<service-name>.<namespace>`的形式，例如`svcb.bar`, CoreDNS会按照 `svcb.bar.foo.svc.cluster.local` -> `svcb.bar.svc.cluster.local` -> `svcb.bar.cluster.local`, 可以看出第二次解析(`svcb.bar.svc.cluster.local`)就会成功


## 访问外部域名是否走search域 ?
测试一下, 进入pod, 使用nslookup指定coreDNS, 查询外部域名`www.trendmicro.com`
```
kubectl -n bar exec -it svcb-5fccb7d86b-wj6nv -- sh
sh-5.1# nslookup www.trendmicro.com 10.96.0.10
Server:         10.96.0.10
Address:        10.96.0.10#53

Non-authoritative answer:
www.trendmicro.com      canonical name = ion.trendmicro.com.edgekey.net.
ion.trendmicro.com.edgekey.net  canonical name = e3576.a.akamaiedge.net.
Name:   e3576.a.akamaiedge.net
Address: 23.217.64.161

抓包
# tcpdump -i eth0 udp
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
13:13:54.687483 IP svcb-5fccb7d86b-wj6nv.37345 > kube-dns.kube-system.svc.cluster.local.domain: 44803+ A? www.trendmicro.com.bar.svc.cluster.local. (58)
13:13:54.687791 IP kube-dns.kube-system.svc.cluster.local.domain > svcb-5fccb7d86b-wj6nv.37345: 44803 NXDomain*- 0/1/0 (151)
13:13:54.688531 IP svcb-5fccb7d86b-wj6nv.47832 > kube-dns.kube-system.svc.cluster.local.domain: 45213+ A? www.trendmicro.com.svc.cluster.local. (54)
13:13:54.688671 IP kube-dns.kube-system.svc.cluster.local.domain > svcb-5fccb7d86b-wj6nv.47832: 45213 NXDomain*- 0/1/0 (147)
13:13:54.688877 IP svcb-5fccb7d86b-wj6nv.56239 > kube-dns.kube-system.svc.cluster.local.domain: 52750+ A? www.trendmicro.com.cluster.local. (50)
13:13:54.689073 IP kube-dns.kube-system.svc.cluster.local.domain > svcb-5fccb7d86b-wj6nv.56239: 52750 NXDomain*- 0/1/0 (143)
13:13:54.689225 IP svcb-5fccb7d86b-wj6nv.41094 > kube-dns.kube-system.svc.cluster.local.domain: 46754+ A? www.trendmicro.com. (36)
13:13:54.720336 IP svcb-5fccb7d86b-wj6nv.36285 > kube-dns.kube-system.svc.cluster.local.domain: 55747+ PTR? 10.0.96.10.in-addr.arpa. (41)
13:13:54.721011 IP kube-dns.kube-system.svc.cluster.local.domain > svcb-5fccb7d86b-wj6nv.36285: 55747*- 1/0/0 PTR kube-dns.kube-system.svc.cluster.local. (116)
13:13:54.731442 IP kube-dns.kube-system.svc.cluster.local.domain > svcb-5fccb7d86b-wj6nv.41094: 46754 3/0/0 CNAME ion.trendmicro.com.edgekey.net., CNAME e3576.a.akamaiedge.net., A 23.208.168.135 (202)
13:13:54.735385 IP svcb-5fccb7d86b-wj6nv.37844 > kube-dns.kube-system.svc.cluster.local.domain: 29753+ AAAA? e3576.a.akamaiedge.net. (40)
13:13:54.775441 IP kube-dns.kube-system.svc.cluster.local.domain > svcb-5fccb7d86b-wj6nv.37844: 29753 0/1/0 (131)
————————————————
```
可以看出, 解析`www.trendmicro.com`走了search域, 有3次无用的DNS请求
`www.trendmicro.com.bar.svc.cluster.local. -> www.trendmicro.com.svc.cluster.local. ->  www.trendmicro.com.cluster.local. -> www.trendmicro.com.`

如果我们只用到了同namespace下的访问、或者跨namespace下的service访问, 可以把ndots默认值改成2, 减少DNS查询, 提高性能


# Kubernetes DNS 策略
在Kubernetes中，dnsPolicy字段定义了Pod的DNS配置策略，提供了四种不同的策略：ClusterFirst、 ClusterFirstWithHostNet、 Default 和 None

### ClusterFirst（默认）
Kubernetes的默认DNS策略, Pod优先使用CoreDNS进行域名解析, 如果CoreDNS无法解析，回退到宿主机的DNS配置进行解析
这是最常见的配置, 适用于需要访问集群内其他服务的应用

### ClusterFirstWithHostNet
这个策略专为使用主机网络(hostNetwork: true）的Pod设计, 仍然优先使用CoreDNS进行解析
适用于需要直接监听宿主机上网络接口, 但仍需访问集群内其他服务的应用

### Default
使用宿主机的DNS设置，完全不使用CoreDNS
适用于主要访问外部服务的应用, 避免CoreDNS解析外部域名带来的延迟问题

### None
完全忽略Kubernetes和宿主机的DNS配置，要求用户自行指定DNS设置。 适用于对DNS配置有高度定制需求的应用. 配置示例如下:
```
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns-pod
spec:
  dnsPolicy: None
  dnsConfig:
    nameservers:
      - 8.8.8.8
      - 8.8.4.4
    searches:
      - ns1.svc.cluster.local
      - mycompany.local
    options:
      - name: ndots
        value: "2"
      - name: edns0
```


## 参考
【1】 [https://coredns.io/manual/toc/](https://coredns.io/manual/toc/)
【2】 [https://cloud.tencent.com/developer/article/2126510](https://cloud.tencent.com/developer/article/2126510)





