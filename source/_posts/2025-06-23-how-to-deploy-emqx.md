---
layout: next
title: EMQX 社区版单机和集群部署
date: 2025-06-03 16:30:25
categories: MQTT
tags: MQTT
---

EMQ 支持 Docker，宿主机，k8s部署；支持单机或集群部署。以下给出EMQX社区版单机和集群部署方法

## 1. Docker单机部署
官方推荐最小配置：2核 4G

### 下载容器镜像
```
docker pull emqx/emqx:5.3.2
```
### 启动容器
```
docker run -d --name emqx \
  -p 1883:1883 \
  -p 8083:8083 \
  -p 8883:8883 \
  -p 8084:8084 \
  -p 18083:18083 \
  emqx/emqx:5.3.2
EMQX 开放端口说明
1883 MQTT端口
8083 MQTT/WebSocket端口
8883 MQTT/SSL端口
8084 MQTT/WebSocket/SSL端口
18083 Dashboard 管理端口
```

<!-- more -->

### 验证
访问 Dashboard, 打开`http://{server_ip}:18083`, 默认账号admin, 密码 public
![](image1.png)

查看容器日志
```
docker logs -f emqx
EMQX 5.3.2 is running now!
```


## 2. 宿主机单机部署
参考：https://blog.csdn.net/m0_65819602/article/details/135876464
测试环境: Rocky Linux 9.5 x86_64 2核 4G内存
以下通过RPM包方式安装部署，通过systemd启动
```
curl -s https://assets.emqx.com/scripts/install-emqx-rpm.sh | sudo bash
yum install emqx -y
systemctl start emqx
```
## 3. 多宿主机集群部署
参考 [官方文档](https://docs.emqx.com/zh/emqx/v5.8/deploy/cluster/create-cluster.html#%E8%87%AA%E5%8A%A8%E9%9B%86%E7%BE%A4%E7%A4%BA%E4%BE%8B%EF%BC%88static-%E6%96%B9%E5%BC%8F%EF%BC%89) 手动创建集群 小节

示例：在两台独立的宿主机上通过静态节点发现方式部署EMQ集群
主机信息
* 主机A: 192.168.149.210
* 主机B: 192.168.149.211

先参考 "宿主机单机部署”，在每个宿主机上部署EMQ
配置每台主机的EMQ
主机A `/etc/emqx/emqx.conf`
```
node {
  name = "emqx@192.168.149.210"
  cookie = "emqxsecretcookie"
  data_dir = "/var/lib/emqx"
}
cluster {
  name = emqx
  discovery_strategy = manual
}
```
主机B `/etc/emqx/emqx.conf`
```
node {
  name = "emqx@192.168.149.211"
  cookie = "emqxsecretcookie"
  data_dir = "/var/lib/emqx"
}
cluster {
  name = emqx
  discovery_strategy = manual
}
```
注：
* node.name 填 emqx@fqdn, fqdn可以是hostname或IP
* cluster.name 填 emqx，和 node.name 前缀保持一致
* discovery_strategy 填 manual, 手动方式加入集群
* 其余配置默认即可

配置完成后，主机B执行如下命令，加入集群
```
emqx ctl cluster join emqx@192.168.149.210
Join the cluster successfully.
Cluster status: #{running_nodes =>
                      ['emqx@192.168.149.210','emqx@192.168.149.211'],
                  stopped_nodes => []}
```
测试，登录dashboard, 查看集群节点创建成功
![](image2.png)

### 节点退出集群
退出集群有以下两种方式：
* 运行 cluster leave 命令：让本节点退出集群。它会通知集群中的其他节点，并停止参与集群操作，在离开之前会完成任何正在进行的任务。
* 运行 cluster force-leave <node@host> 命令：在集群内移除节点。目标节点将被强制从集群中移除。当节点出现故障或无响应时，通常使用此命令。

例如，在之前构建的集群中，如果emqx@192.168.149.211想要离开集群，您可以在192.168.149.211上运行以下命令
```
emqx ctl cluster leave
```
或者在其他节点上运行以下命令来从集群中移除192.168.149.211
```
emqx ctl cluster force-leave emqx@192.168.149.211
```
查看集群状态
```
emqx_ctl cluster status
```
