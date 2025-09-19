---
layout: next
title: k8s 部署 EMQX 5.8.6 静态三节点集群
date: 2025-09-19 10:23:50
categories: MQTT
tags: MQTT
---

## 一、前置操作：创建命名空间

先为 EMQX 单独创建命名空间，隔离资源，避免与其他服务冲突：

```
kubectl create ns emqx
```

## 二、核心配置文件：emqx-cluster.yaml

核心配置包含 **Headless Service、NodePort Service、ConfigMap、StatefulSet** 四部分，需重点关注集群通信、端口映射、存储与部署策略。

<!-- more -->

### 1. 配置文件完整内容

```
apiVersion: v1
kind: Service
metadata:
  name: emqx-headless
  namespace: emqx
spec:
  selector:
    app: emqx
  clusterIP: None  # Headless 服务，无集群IP，用于集群内部通信
  publishNotReadyAddresses: true  # 允许未就绪节点被发现，保障集群启动
  ports:
    - name: mqtt
      port: 1883
    - name: ws
      port: 8083
    - name: cluster
      port: 4370  # EMQX 集群通信端口
    - name: epmd
      port: 5369  # Erlang 节点发现端口
---
apiVersion: v1
kind: Service
metadata:
  name: emqx-nodeport
  namespace: emqx
spec:
  selector:
    app: emqx
  type: NodePort  # 暴露节点端口，供外部访问
  ports:
    - name: mqtt
      port: 1883
      targetPort: 1883
      nodePort: 31883  # 外部访问 MQTT TCP 端口
    - name: ws
      port: 8083
      targetPort: 8083
      nodePort: 30083  # 外部访问 MQTT WebSocket 端口
    - name: dashboard
      port: 18083
      targetPort: 18083
      nodePort: 31083  # 外部访问 EMQX 管理页面端口
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: emqx-config
  namespace: emqx
data:
  emqx.conf: |
    node {
      data_dir = "/opt/emqx/data"  # 数据存储目录
    }

    cluster {
      name = emqxcl  # 集群名称
      discovery_strategy = static  # 静态集群发现（核心，指定节点列表）
      autoheal = true  # 集群自动修复
      static {
        # 静态节点列表，格式：emqx@<pod名>.<headless服务名>.<命名空间>.svc.cluster.local
        seeds = ["emqx@emqx-0.emqx-headless.emqx.svc.cluster.local", 
                 "emqx@emqx-1.emqx-headless.emqx.svc.cluster.local", 
                 "emqx@emqx-2.emqx-headless.emqx.svc.cluster.local"]
      }
    }

    # MQTT 基础配置（端口、连接数等，保持默认即可）
    mqtt { max_packet_size = 268435455 }
    listeners.tcp.default { bind = 1883; max_connections = 1024 }
    listeners.ws.default { enable = true; bind = 8083; max_connections = 1024 }
    listeners.ssl.default { enable = false }
    listeners.wss.default { enable = false }

    # 管理页面配置
    dashboard { listeners.http { bind = 18083 } }
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: emqx
  namespace: emqx
spec:
  serviceName: emqx-headless  # 关联 Headless 服务，保障固定网络标识
  replicas: 3  # 三节点集群
  selector:
    matchLabels:
      app: emqx
  updateStrategy:
    type: RollingUpdate  # 滚动更新（更新顺序：2→1→0）
  template:
    metadata:
      labels:
        app: emqx
    spec:
      # 亲和性配置：避免同一节点部署多个 EMQX 实例，保证高可用
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - emqx
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: emqx
        imagePullPolicy: IfNotPresent
        image: "mydockerregistry.com:5000/emqx:5.8.6"  # 替换为你的 EMQX 镜像地址
        ports:
        - containerPort: 1883; name: mqtt
        - containerPort: 8083; name: ws
        - containerPort: 18083; name: dashboard
        - containerPort: 4370; name: cluster
        - containerPort: 5369; name: epmd
        # 环境变量：配置节点名称、Cookie、管理页面账号密码
        env:
        - name: POD_NAME
          valueFrom: { fieldRef: { fieldPath: metadata.name } }
        - name: EMQX_NODE__NAME
          value: "emqx@$(POD_NAME).emqx-headless.emqx.svc.cluster.local"  # 动态生成节点名
        - name: EMQX_NODE__COOKIE
          value: "emqxsecretcookie"  # 集群 Cookie（所有节点需一致）
        - name: EMQX_DASHBOARD__DEFAULT_USERNAME
          value: "admin"  # 管理页面默认用户名
        - name: EMQX_DASHBOARD__DEFAULT_PASSWORD
          value: "password@123"  # 管理页面默认密码（建议部署后修改）
        # 挂载配置与存储
        volumeMounts:
        - name: emqx-data
          mountPath: /opt/emqx/data  # 数据持久化目录
        - name: emqx-config
          mountPath: /opt/emqx/etc/emqx.conf
          subPath: emqx.conf  # 挂载 ConfigMap 中的 emqx.conf
        # 健康检查：确保实例正常运行
        livenessProbe:
          tcpSocket: { port: 1883 }
          initialDelaySeconds: 60  # 启动60秒后开始检查
          periodSeconds: 10  # 每10秒检查一次
          timeoutSeconds: 5
          failureThreshold: 3
      volumes:
      - name: emqx-config
        configMap: { name: emqx-config }  # 关联 ConfigMap
  podManagementPolicy: OrderedReady  # 顺序启动（启动顺序：0→1→2）
  # 存储声明模板：为每个节点分配独立存储
  volumeClaimTemplates:
  - metadata:
      name: emqx-data
    spec:
      accessModes: [ "ReadWriteOnce" ]  # 单节点读写
      resources:
        requests: { storage: 2Gi }  # 每个节点申请2Gi存储
      storageClassName: "local-path"  # 存储类（需提前创建，如 local-path 或云存储）
```

## 三、部署与验证

### 1. 应用配置文件

执行部署命令，加载 `emqx-cluster.yaml` 配置：

```
kubectl apply -f emqx-cluster.yaml
```

### 2. 验证部署状态

#### （1）查看 Pod 状态（需全部为 Running）

```
kubectl -n emqx get pods -o wide
```

示例输出（三节点均 Running 表示启动成功）：

```
NAME     READY   STATUS    RESTARTS      AGE   IP            NODE        NOMINATED NODE

emqx-0   1/1     Running   1 (27m ago)   15h   10.42.3.196   k3s-node3   \<none>

emqx-1   1/1     Running   1 (28m ago)   15h   10.42.0.245   k3s-node1   \<none>

emqx-2   1/1     Running   1 (27m ago)   15h   10.42.1.200   k3s-node2   \<none>
```

#### （2）查看存储声明（需全部为 Bound）

```
kubectl -n emqx get pvc
```

示例输出（存储均 Bound 表示持久化配置生效）：

```
NAME               STATUS   VOLUME                                     CAPACITY   ACCESS MODES

emqx-data-emqx-0   Bound    pvc-caea27ef-28a9-4d68-9c5a-0bdedb3cf819   2Gi        RWO

emqx-data-emqx-1   Bound    pvc-f4e575df-0411-4892-a6ad-af5adabe8243   2Gi        RWO

emqx-data-emqx-2   Bound    pvc-17550095-4775-4c59-aade-c095872b0348   2Gi        RWO
```

#### （3）查看 Service 状态（确认端口映射）

```
kubectl -n emqx get svc -o wide
```

示例输出（NodePort 端口 31883/30083/31083 需正常显示）：

```
NAME            TYPE        CLUSTER-IP     PORT(S)                                         AGE

emqx-headless   ClusterIP   None           1883/TCP,8083/TCP,4370/TCP,5369/TCP             15h

emqx-nodeport   NodePort    10.43.96.210   1883:31883/TCP,8083:30083/TCP,18083:31083/TCP   15h
```

## 四、关键说明与注意事项

**端口用途**：

*   1883（TCP）/8083（WebSocket）：MQTT 客户端连接端口；

*   18083：EMQX 管理页面端口（访问地址：`http://k8s-node-ip:31083`）；

*   4370/5369：EMQX 集群内部通信端口（无需外部暴露）。

**外部访问建议**：

*   MQTT TCP（31883）：可前置 Nginx/Haproxy 做四层负载均衡；

*   MQTT WebSocket（30083）：可前置 Nginx/Haproxy/Ingress 做七层负载均衡。

**安全优化**：

*   管理页面密码（`password@123`）需部署后立即修改；

*   镜像地址（`mydockerregistry.com:5000/emqx:5.8.6`）替换为实际可用的镜像（如官方镜像 `emqx/emqx:5.8.6`）。

**部署脚本参考**：

完整脚本可查看 GitHub 仓库：[https://github.com/PCJ600/emqx-deploy](https://github.com/PCJ600/emqx-deploy)
