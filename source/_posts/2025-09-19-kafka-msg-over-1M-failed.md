---
layout: next
title: Confluent-Kafka-go 发布超过 1M 消息失败问题解决
date: 2025-09-19 17:23:17
categories: Kafka
tags: Kafka
---

## 问题描述

cp-kafka 4.0.0 集群默认单条消息上限为 1M（message.max.bytes 默认值 1048576），confluent-kafka-go 客户端若未调整对应配置，发布超过 1M 的消息会触发两类报错：
* 客户端：Local: Message size too large（本地检测超上限）；​
* 集群：Broker: Message size too large（Broker 拒绝接收）。

## 临时解决方案：客户端与Kafka集群配置对齐

修改发布端的 message.max.bytes 参数，同时同步调整消费端的 fetch.message.max.bytes 参数，避免后续消费时因消息大小超限导致接收失败。示例代码如下：
<!-- more -->

```
// Creates a new KafkaClient instance.
func New(cfg MQConfig) *KafkaClient {
    maxPktBytes := 10485760 // 10M
    return &KafkaClient{
        pConfig: &kafka.ConfigMap{
            "bootstrap.servers": cfg.Brokers,
            "acks":              "all",
            "retries":           3,
            // 设置发布端允许的最大消息大小为10M，需与Kafka集群配置对齐
            "message.max.bytes": maxPktBytes,
        },
        cConfig: &kafka.ConfigMap{
            "bootstrap.servers":                  cfg.Brokers,
            "group.id":                           cfg.ConsumerGroup,
			// ...
            // 消费者需同步调整，避免接收时超上限
            "fetch.message.max.bytes":            maxPktBytes,
        },
		// ...
    }
}
```
## cp-kafka 集群配置调整

通过环境变量调整集群消息上限，确保与客户端一致：
```
# cp-kafka 容器环境变量（K8s/Docker部署）​
env:​
  # 集群接收消息的最大大小，与客户端 message.max.bytes 一致​
  - name: KAFKA_MESSAGE_MAX_BYTES​
    value: "10485760" # 示例为10M，需与客户端配置匹配​
  # 副本同步上限需略大，避免元数据占用空间导致同步失败​
  - name: KAFKA_REPLICA_FETCH_MAX_BYTES​
    value: "12582912" # 建议为消息上限的1.2倍
```
配置修改后，需重启 cp-kafka 集群容器，确保配置生效。​


注: 放大 message.max.bytes 会占用更多集群带宽与磁盘，增加 Broker 压力。 生产环境不推荐修改默认配置，客户端需要将大消息做拆分