---
layout: next
title: 在 k8s 上部署 Kafka 4.0 3节点集群
date: 2025-09-19 15:11:06
categories: Kafka
tags: Kafka
---

本文详细介绍如何在 K3s 环境中部署一个 3 节点的 Apache Kafka 4.0.0 集群（对应 Confluent Platform 8.0.0），使用 KRaft 模式消除对 ZooKeeper 的依赖。

## 镜像与版本选择

​​confluentinc/cp-kafka:8.0.0​​：Confluent 官方镜像，提供企业级特性
Kafka 4.0.x 移除了对 ZooKeeper 的依赖（KRaft 模式 GA），带来显著性能提升和运维简化
```
docker pull confluentinc/cp-kafka:8.0.0
```

## Kafka集群部署架构

* 3节点Kafka集群​​：利用 StatefulSet 的稳定网络标识（kafka-0、kafka-1、kafka-2）
* Pod反亲和性​​：确保3个副本分散在不同k3s节点（221、222、223）
* KRaft模式​​：无需ZooKeeper，简化部署架构
* 数据持久化​​：每个节点使用独立PVC存储数据

<!-- more -->

## k8s配置文件

Kafka集群StatefulSet, Service配置
```
# kafka-cluster-3nodes.yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka-headless
  namespace: kafka
  labels:
    app: kafka
spec:
  selector:
    app: kafka
  ports:
  - name: plaintext
    port: 9092
    targetPort: 9092
  - name: controller
    port: 9093
    targetPort: 9093
  clusterIP: None  # Headless Service用于Pod直接通信
  publishNotReadyAddresses: true  # 允许访问未就绪的Pod
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
  namespace: kafka
spec:
  serviceName: kafka-headless  # 关联Headless Service
  replicas: 3  # 3节点集群
  selector:
    matchLabels:
      app: kafka
  updateStrategy:
    type: RollingUpdate  # 滚动更新策略
  template:
    metadata:
      labels:
        app: kafka
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - kafka
            topologyKey: "kubernetes.io/hostname"  # 确保Pod分散在不同节点
      containers:
      - name: kafka
        image: mydockerregistry.com:5000/confluentinc/cp-kafka:8.0.0
        imagePullPolicy: IfNotPresent
        command: ["/bin/sh", "-c"]
        args:
          - |
            export KAFKA_NODE_ID=${HOSTNAME##*-}  # 从Pod名称提取节点ID
            echo "Starting Kafka with NODE_ID: $KAFKA_NODE_ID"
            exec /etc/confluent/docker/run
        ports:
          - name: plaintext
            containerPort: 9092  # 客户端通信端口
          - name: controller
            containerPort: 9093  # 控制器通信端口
          - name: external
            containerPort: 31092  # 外部访问端口
            hostPort: 31092  # 使用hostPort暴露到节点
        env:
          # 关键环境变量配置
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: K8S_NODE_IP
            valueFrom:
              fieldRef:
                fieldPath: status.hostIP  # 获取节点IP用于外部访问
          - name: CLUSTER_ID
            value: "LTI4QjFBNTcwNTJENDM2Qk"  # 集群唯一标识
          - name: KAFKA_HEAP_OPTS
            value: "-Xms512m -Xmx512m"  # JVM内存配置
          - name: KAFKA_PROCESS_ROLES
            value: "broker,controller"  # 节点同时担任broker和controller
          - name: KAFKA_CONTROLLER_LISTENER_NAMES
            value: "CONTROLLER"
          - name: KAFKA_LISTENERS
            value: "PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093,EXTERNAL://0.0.0.0:31092"
          - name: KAFKA_LISTENER_SECURITY_PROTOCOL_MAP
            value: "CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,EXTERNAL:PLAINTEXT"
          - name: KAFKA_ADVERTISED_LISTENERS
            # 内部使用Pod DNS，外部使用节点IP
            value: "PLAINTEXT://$(POD_NAME).kafka-headless.kafka:9092,EXTERNAL://$(K8S_NODE_IP):31092"
          - name: KAFKA_INTER_BROKER_LISTENER_NAME
            value: "PLAINTEXT"
          - name: KAFKA_CONTROLLER_QUORUM_VOTERS
            # 配置3个控制器投票器
            value: "0@kafka-0.kafka-headless.kafka:9093,1@kafka-1.kafka-headless.kafka:9093,2@kafka-2.kafka-headless.kafka:9093"
          - name: KAFKA_LOG_DIRS
            value: "/var/lib/kafka/data"
          - name: KAFKA_MESSAGE_MAX_BYTES
            value: "10485760"  # 10MB最大消息大小
          - name: KAFKA_REPLICA_FETCH_MAX_BYTES
            value: "10485760"
        livenessProbe:
          tcpSocket:
            port: 9092  # 仅保留存活探针，去掉就绪探针
          initialDelaySeconds: 60  # 给予足够启动时间
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        volumeMounts:
        - name: kafka-data
          mountPath: /var/lib/kafka/data
  volumeClaimTemplates:
    - metadata:
        name: kafka-data
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: "local-path"  # 使用k3s默认的local-path存储类
        resources:
          requests:
            storage: 4Gi  # 每个节点4GB存储
```

Kafka主题创建Job配置

```
# kafka-topics-create.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: kafka-topics-create
  namespace: kafka
spec:
  backoffLimit: 100  # 允许重试多次
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: create-topics
        image: mydockerregistry.com:5000/confluentinc/cp-kafka:8.0.0
        command:
          - sh
          - -c
          - |
            # 等待所有Kafka节点就绪
            KAFKA_NODES="kafka-0.kafka-headless.kafka:9092 kafka-1.kafka-headless.kafka:9092 kafka-2.kafka-headless.kafka:9092"
            
            echo "Waiting for all Kafka nodes to be ready..."
            for node in $KAFKA_NODES; do
              until kafka-topics --list --bootstrap-server $node > /dev/null 2>&1; do
                echo "Waiting for $node..."
                sleep 10
              done
            done

            # 创建业务所需主题
            kafka-topics --create --bootstrap-server kafka-0.kafka-headless.kafka:9092 --topic up.egw.telemetry \
              --partitions 6 --replication-factor 3 --config retention.ms=3600000 --if-not-exists
            kafka-topics --create --bootstrap-server kafka-0.kafka-headless.kafka:9092 --topic up.egw.model \
              --partitions 6 --replication-factor 3 --config retention.ms=604800000 --if-not-exists
            kafka-topics --create --bootstrap-server kafka-0.kafka-headless.kafka:9092 --topic up.egw.alert \
              --partitions 6 --replication-factor 3 --config retention.ms=604800000 --if-not-exists
            kafka-topics --create --bootstrap-server kafka-0.kafka-headless.kafka:9092 --topic up.egw.notify \
              --partitions 6 --replication-factor 3 --config retention.ms=3600000 --if-not-exists
            kafka-topics --create --bootstrap-server kafka-0.kafka-headless.kafka:9092 --topic down.egw.notify \
              --partitions 6 --replication-factor 3 --config retention.ms=3600000 --if-not-exists
            kafka-topics --create --bootstrap-server kafka-0.kafka-headless.kafka:9092 --topic audit \
              --partitions 1 --replication-factor 3 --config retention.ms=604800000 --if-not-exists
            echo "All topics created"
```

## 部署命令

```
# 创建命名空间
kubectl create ns kafka

# 部署Kafka集群
kubectl apply -f kafka-cluster-3nodes.yaml

# 等待集群就绪后创建主题
kubectl apply -f kafka-topics-create.yaml

# 检查部署状态
kubectl get pods -n kafka -w
kubectl logs -n kafka -l app=kafka -f
```

## 部署遇到问题和解决方法

### 问题1: 集群启动失败，Pod不Ready, 集群就绪探针阻碍初始化

​​原因​​：Kafka KRaft 集群在初始化阶段需要节点互相发现才能监听端口，就绪探针会误判节点未就绪
​​解决方案​​：临时移除就绪探针，仅保留存活探针，并增加初始延迟

### 问题2: 所有Kafka节点的外部地址(EXTERNAL)配置为同一个地址时, 从外部订阅会失败, 但发布正常
​​原因​​：Kafka 的 advertised.listeners必须与客户端实际连接地址严格匹配, 3个节点应该配3个独立外部地址
​​解决方案​​：为每个节点配置独立的外部访问地址 (仅临时测试用, 生产环境不提供外部地址)

```
# 使用hostPort为每个Pod暴露独立端口
ports:
- name: external
  containerPort: 31092
  hostPort: 31092  # 直接绑定到节点端口

# 广告监听器使用节点IP和固定端口
- name: KAFKA_ADVERTISED_LISTENERS
  value: "PLAINTEXT://$(POD_NAME).kafka-headless.kafka:9092,EXTERNAL://$(K8S_NODE_IP):31092"
```

内部访问：使用 kafka-0.kafka-headless.kafka:9092等内部DNS
外部访问：使用 k8s节点IP:NodePort(31092)连接指定的broker

## 部署脚本参考：

完整脚本可查看 GitHub 仓库：https://github.com/PCJ600/kafka-deploy