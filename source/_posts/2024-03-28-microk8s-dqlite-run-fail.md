---
layout: next
title: 'Microk8s dqlite启动失败, raft_start(): io: load closed segment XXX: entries count in preamble is zero'
date: 2024-03-28 20:13:20
categories: troubleshooting
tags:
- k8s
- troubleshooting
---

## 定位步骤
后台查看Microk8s相关进程：
```
   7281 ?        Ss    27:02 /bin/bash /var/lib/snapd/snap/microk8s/3699/apiservice-kicker
 749807 ?        S      0:00  \_ sleep 5
   7290 ?        Ss     0:00 /bin/bash /var/lib/snapd/snap/microk8s/3699/run-cluster-agent-with-args
   7362 ?        S      2:32  \_ python3 /snap/microk8s/3699/usr/bin/gunicorn3 cluster.agent:app --bind 0.0.0.0:25000 --keyfile /var/snap/microk8s/3699/certs/server.key --certfile /var/snap/microk8s/3699/certs/server.crt --timeout 240
   7684 ?        S      0:02      \_ python3 /snap/microk8s/3699/usr/bin/gunicorn3 cluster.agent:app --bind 0.0.0.0:25000 --keyfile /var/snap/microk8s/3699/certs/server.key --certfile /var/snap/microk8s/3699/certs/server.crt --timeout 240
   7310 ?        Ssl  399:16 /snap/microk8s/3699/bin/containerd --config /var/snap/microk8s/3699/args/containerd.toml --root /var/snap/microk8s/common/var/lib/containerd --state /var/snap/microk8s/common/run/containerd --address /var/snap/microk8s/common/run/containerd.sock
```
和正常环境对比，发现`k8s-dqlite`, `kubelite`进程执行异常，如下：
<!-- more -->
```
   7457 ?        Ssl  752:10 /snap/microk8s/3699/bin/k8s-dqlite --storage-dir=/var/snap/microk8s/3699/var/kubernetes/backend/ --listen=unix:///var/snap/microk8s/3699/var/kubernetes/backend/kine.sock:12379
   7489 ?        Ssl  1777:58 /snap/microk8s/3699/kubelite --scheduler-args-file=/var/snap/microk8s/3699/args/kube-scheduler --controller-manager-args-file=/var/snap/microk8s/3699/args/kube-controller-manager --proxy-args-file=/var/snap/microk8s/3699/args/kube-proxy --kubelet-args-file=/var/snap/microk8s/3699/args/kubelet --apiserver-args-file=/var/snap/microk8s/3699/args/kube-apiserver
```
通过`journactl`查看错误日志:
fatal msg="Failed to start server: start node: raft_start(): io: **load closed segment 0000000014232389-0000000014232597**: entries batch 394 starting at byte 8384000: entries count in preamble is zero\n"
Feb 03 08:18:08 GIFOCTMSG02 systemd[1]: snap.microk8s.daemon-k8s-dqlite.service: Main process exited, code=exited, status=1/FAILURE

根据关键字查Google找到解决方案：[https://discuss.linuxcontainers.org/t/error-failed-to-start-dqlite-server-raft-start/6931](https://discuss.linuxcontainers.org/t/error-failed-to-start-dqlite-server-raft-start/6931)

## 解决方法
根据错误日志中"segment **0000000014232389-0000000014232597**"的信息，找到出错的文件`/var/snap/microk8s/current/var/kubernetes/backend/0000000014232389-0000000014232597`，删除此文件后重启microk8s，发现问题解决。文件出错原因可能是因为机器被异常下电，导致文件损坏
```
microk8s stop
rm /var/snap/microk8s/current/var/kubernetes/backend/0000000014232389-0000000014232597
microk8s start
```
`/var/snap/microk8s/3699/var/kubernetes/backend`的文件如下：
```
# ll /var/snap/microk8s/3699/var/kubernetes/backend
total 153212
-rw-rw---- 1 root microk8s 8381624 Jan 28 00:11 0000000014225187-0000000014225660
-rw-rw---- 1 root microk8s 8386880 Jan 28 00:12 0000000014225661-0000000014226264
-rw-rw---- 1 root microk8s 8386016 Jan 28 00:16 0000000014226265-0000000014226628
-rw-rw---- 1 root microk8s 8381552 Jan 28 00:16 0000000014226629-0000000014227272
-rw-rw---- 1 root microk8s 8378744 Jan 28 00:19 0000000014227273-0000000014227706
-rw-rw---- 1 root microk8s 8362544 Jan 28 00:21 0000000014227707-0000000014228257
-rw-rw---- 1 root microk8s 8377520 Jan 28 00:23 0000000014228258-0000000014228788
-rw-rw---- 1 root microk8s 8386592 Jan 28 00:26 0000000014228789-0000000014229217
-rw-rw---- 1 root microk8s 8368952 Jan 28 00:26 0000000014229218-0000000014229857
-rw-rw---- 1 root microk8s 8361608 Jan 28 00:30 0000000014229858-0000000014230224
-rw-rw---- 1 root microk8s 8387384 Jan 28 00:31 0000000014230225-0000000014230835
-rw-rw---- 1 root microk8s 8377232 Jan 28 00:34 0000000014230836-0000000014231305
-rw-rw---- 1 root microk8s 8387384 Jan 28 00:36 0000000014231306-0000000014231802
-rw-rw---- 1 root microk8s 8373272 Jan 28 00:37 0000000014231803-0000000014232388
-rw-rw---- 1 root microk8s 8384512 Jan 28 00:41 0000000014232389-0000000014232597
-rw-rw---- 1 root microk8s      80 Jan 28 00:40 0000000014232598-0000000014232598
-rw-rw---- 1 root microk8s    1883 Dec 14 03:38 cluster.crt
-rw-rw---- 1 root microk8s    3272 Dec 14 03:38 cluster.key
-rw-rw---- 1 root microk8s      63 Feb  4 03:11 cluster.yaml
-rw-rw-r-- 1 root microk8s       2 Feb  4 03:12 failure-domain
-rw-rw---- 1 root microk8s      57 Dec 14 03:38 info.yaml
-rw-rw-r-- 1 root microk8s       0 Feb  2 10:24 kine.sock
-rw-rw---- 1 root microk8s      63 Dec 19 07:56 localnode.yaml
-rw-rw---- 1 root microk8s      32 Dec 14 03:38 metadata1
-rw-rw---- 1 root microk8s 8388608 Jan 28 00:41 open-28053
-rw-rw---- 1 root microk8s 8388608 Jan 28 00:41 open-28054
-rw-rw---- 1 root microk8s 8388608 Jan 28 00:45 open-28055
-rw-rw---- 1 root microk8s 3069878 Jan 28 00:40 snapshot-1-14232647-3429895302
-rw-rw---- 1 root microk8s      72 Jan 28 00:40 snapshot-1-14232647-3429895302.meta
-rw-rw---- 1 root microk8s 2893883 Jan 28 00:43 snapshot-1-14233671-3430050282
-rw-rw---- 1 root microk8s      72 Jan 28 00:43 snapshot-1-14233671-3430050282.meta
```
每个文件的作用：
* 数据段文件（如：0000000014225187-0000000014225660）：
这些文件包含 dqlite 存储的实际数据段。每个文件代表一个数据段，存储着相应的数据。这些文件在 dqlite 数据库引擎中用于持久性地存储数据。
* 索引段文件（如：0000000014225661-0000000014226264）：
类似于数据段文件，这些文件包含 dqlite 存储的索引段，用于加速数据检索。
* 快照文件（如：snapshot-1-14232647-3429895302）：
这些文件包含数据库在某个时间点的快照，用于还原数据库状态。在备份和还原过程中，这些文件可能会用到。
* 信息文件（info.yaml）：
这是一个 YAML 格式的文件，包含有关 dqlite 数据库的一些元信息。
* 配置文件（cluster.yaml）：
包含 dqlite 集群的配置信息，例如节点地址、端口等。
* 证书文件（cluster.crt、cluster.key）：
这是用于在 dqlite 集群中进行安全通信的证书文件。
* 本地节点配置文件（localnode.yaml）：
包含本地节点的配置信息，例如节点 ID。
* 元数据文件（metadata1）：
存储 dqlite 内部使用的一些元数据信息。
* 打开文件（如：open-28055）：
这些文件可能是当前正在使用的打开文件。它们通常在 dqlite 服务正在运行时存在。
* 失败域文件（failure-domain）：
可能包含关于节点失败域的信息，用于高可用性和容错。

## 参考
[https://discuss.linuxcontainers.org/t/error-failed-to-start-dqlite-server-raft-start/6931](https://discuss.linuxcontainers.org/t/error-failed-to-start-dqlite-server-raft-start/6931)
