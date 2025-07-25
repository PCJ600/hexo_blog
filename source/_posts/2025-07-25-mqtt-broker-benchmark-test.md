---
layout: next
title: EMQX 性能测试示例
date: 2025-07-25 08:33:39
categories: MQTT
tags: MQTT
---

## 前言
对 EMQX 5.8.6 (社区版)  做性能测试，包括并发连接数测试和吞吐量测试

## 测试场景
- 多对1发布订阅 (**大量边端设备上报高频采样数据到云端, 做性能测试**)
- 1对1发布订阅 (云端下发到边端设备的控制指令，低频数据，不做测试)
- 1对多发布订阅 (云端->边端的通信，低频数据，不做测试)

<!-- more -->

## 测试拓扑
![](test_topo.png)

## 测试机器
EMQX Broker 服务端：
* Intel(R) Xeon(R) Gold 5218 CPU @ 2.30GHz
* Debian 12 或 Rocky Linux 9.6 x86_64
* 32G 内存

压力机：
* 4核，4G内存
* Debian 12 或 Rocky Linux 9.6 x86_64
* 安装 emqtt-bench v0.5.2 测试工具

## 服务端和压力机执行系统参数调优
配置ulimit等系统参数, [参考文档](https://docs.emqx.com/zh/emqx/latest/performance/tune.html)

## 安装 emqtt-bench 性能测试工具
每台压力机安装 emqtt-bench 0.5.2

```
# 先安装依赖 erl 27.2
wget https://github.com/rabbitmq/erlang-rpm/releases/download/v27.2/erlang-27.2-1.el9.x86_64.rpm
yum install -y erlang-27.2-1.el9.x86_64.rpm
erl -version

# 安装 emqtt-bench
wget https://github.com/emqx/emqtt-bench/releases/download/0.5.2/emqtt-bench-0.5.2-el9-amd64.tar.gz
tar xfz emqtt-bench-0.5.2-el9-amd64.tar.gz
cd bin/
./emqtt_bench
Usage: emqtt_bench pub | sub | conn [--help]
```

emqtt_bench 基本用法如下：
```
# 订阅
./emqtt_bench sub -t "\$share/group1/device/+/telemetry" -c 2 -h emqx -u "test" -P "password@123"

# 发布
./emqtt_bench pub -t "device/%i/telemetry" -c 10000 -h emqx -s 1024 -m "$(base64 /dev/urandom | head -c 1024)" -u "test" -P "password@123"
```

参数说明如下：

| 参数 | 描述 |
| --- | ------------------ |
| -R | 新连接创建速率，即每秒新增的客户端数 |
| -c | 总客户端连接数 |
| -I | 每个客户端发布消息间隔 (默认值： 1000 ms) |
| -s | 指定消息大小 (单位：字节) |


## Docker compose 部署单节点 EMQX 5.8.6
新增 docker-compose.yml
```
services:
  emqx:
    image: swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/emqx/emqx:5.8.6
    container_name: emqx
    hostname: emqx-node1.cluster.local
    ports:
      - "1883:1883"
      - "18083:18083"
    volumes:
      - emqx_data:/opt/emqx/data
      - ./emqx.conf:/opt/emqx/etc/emqx.conf
    environment:
      - EMQX_HOST=emqx-node1.cluster.local
      - EMQX_CLUSTER__DISCOVERY_STRATEGY=static
      - EMQX_DASHBOARD__DEFAULT_USERNAME=admin
      - EMQX_DASHBOARD__DEFAULT_PASSWORD=password@123
    privileged: true
    restart: on-failure:3
    healthcheck:
      test: ["CMD", "/opt/emqx/bin/emqx_ctl", "status"]
      interval: 60s
      timeout: 15s
      retries: 3
      start_period: 90s
    sysctls:
      net.core.somaxconn: 32768
      net.ipv4.tcp_syncookies: 1
      net.ipv4.tcp_max_syn_backlog: 16384
      net.ipv4.tcp_max_tw_buckets: 1048576
      net.ipv4.tcp_fin_timeout: 15
      net.ipv4.ip_local_port_range: "1024 65535"
    ulimits:
      nproc: 1048576
      nofile:
        soft: 1048576
        hard: 1048576

    networks:
      - demo_network

volumes:
  emqx_data:

networks:
  demo_network:
    driver: bridge
```

创建 emqx.conf (和 docker-compose.yml 同一个目录下)
```
node {
  name = "emqx@127.0.0.1"
  cookie = "emqxsecretcookie"
  data_dir = "data"
}

cluster {
  name = emqxcl
  discovery_strategy = manual
}

dashboard {
    listeners {
        http.bind = 18083
        # https.bind = 18084
        https {
            ssl_options {
                certfile = "${EMQX_ETC_DIR}/certs/cert.pem"
                keyfile = "${EMQX_ETC_DIR}/certs/key.pem"
            }
        }
    }
}

mqtt {
  mqueue_store_qos0 = false
}

force_shutdown {
  enable = false
}
```

启动 EMQX
```
docker compose up -d
```


查看 EMQX 空载 CPU和内存使用率：1.8%、290M
![](image1.png)

## 并发连接数测试

### 用例一：测试2000个并发连接 (2000个发布者，每分钟发布1条消息，2个订阅者，消息大小：1024字节，qos 0)

```
# 订阅端执行
./emqtt_bench sub -t "\$share/group1/device/+/telemetry" -c 2 -h emqx-node1.cluster.local -u "test" -P "password@123"

# 发布端执行
./emqtt_bench pub -t "device/%i/telemetry" -c 2000 -I 60000 -L 6000 -h emqx-node1.cluster.local -s 1024 -m "$(base64 /dev/urandom | head -c 1024)" -u "test" -P "password@123"
```

#### emqtt-bench 测试工具统计

![](image1_1.png)

* 发布速率：65 msg/s, 订阅速率：65 msg/s
* 消息发布数：6000，消息订阅数：6000，消息丢失数：0

#### EMQX 官方看板统计数据

![](image1_2.png)

* 连接数：2002，发布速率：102 msg/s，订阅速率：102 msg/s

#### CPU、内存使用率统计

**EMQX服务端**
![](image1_3.png)

**订阅端**
![](image1_4.png)

* EMQ 服务端：312%、2.5G
* 订阅端：26.7%、610M


### 用例二：测试5w个并发连接 (5w个发布者，每分钟发布1条消息，2个订阅者，消息大小：1024字节，qos 0)

```
# 订阅端执行
./emqtt_bench sub -t "\$share/group1/device/+/telemetry" -c 2 -h emqx-node1.cluster.local -u "test" -P "password@123"

# 发布端执行
./emqtt_bench pub -t "device/%i/telemetry" -c 50000 -R 500 -I 60000 -L 300000 -h emqx-node1.cluster.local -s 1024 -m "$(base64 /dev/urandom | head -c 1024)" -u "test" -P "password@123"
```

#### emqtt-bench 测试工具统计
![](image2_1.png)

* 发布速率：966 msg/s, 订阅速率：933 msg/s
* 消息发布数：30万，消息订阅数：30万，消息丢失数：0

#### EMQX 官方看板统计数据
![](image2_2.png)

* 连接数：50002，发布速率：965 msg/s，订阅速率：965 msg/s

#### CPU、内存使用率统计

**EMQ负载**
![](image2_3.png)

**订阅端负载**
![](image2_4.png)

* EMQ 服务端：220%、1.3G
* 订阅端：26.7%、610M


### 用例三：测试10w个并发连接 (10w个发布者，每分钟发布1条消息，2个订阅者，消息大小：1024字节，qos 0)

```
# 订阅端执行
./emqtt_bench sub -t "\$share/group1/device/+/telemetry" -c 2 -h emqx-node1.cluster.local -u "test" -P "password@123"

# 发布端执行 (两台压力机同时执行)
./emqtt_bench pub -t "device/%i/telemetry" -c 50000 -R 250 -I 60000 -L 300000 -h emqx-node1.cluster.local -s 1024 -m "$(base64 /dev/urandom | head -c 1024)" -u "test" -P "password@123"
```

#### emqtt-bench 测试工具统计
![](image3_1.png)

* 发布速率：1967 msg/s, 订阅速率：1967 msg/s
* 消息发布数：60万，消息订阅数：60万，消息丢失数：0

#### EMQX 官方看板统计数据
![](image3_2.png)

* 连接数：100002，发布速率：1487 msg/s，订阅速率：1487 msg/s

#### CPU、内存使用率统计

**EMQ负载**
![](image3_3.png)

**订阅端负载**
![](image3_4.png)

* EMQ 服务端：312%、2.5G
* 订阅端：26.7%、610M

### 用例四：测试单节点最大并发连接数 (M个发布者，每分钟发布1条消息，2个订阅者，消息大小：1024字节，qos 0)

给压力机配10个虚拟IP （每个IP测5万个连接，最多可以测50万个并发连接）

```
# 给压力机网卡添加10个虚拟IP, 1个IP测5万个连接
ip addr add 10.17.14.210/24 dev ens192
ip addr add 10.17.14.211/24 dev ens192
ip addr add 10.17.14.212/24 dev ens192
ip addr add 10.17.14.213/24 dev ens192
ip addr add 10.17.14.214/24 dev ens192
ip addr add 10.17.14.215/24 dev ens192
ip addr add 10.17.14.216/24 dev ens192
ip addr add 10.17.14.217/24 dev ens192
ip addr add 10.17.14.218/24 dev ens192
ip addr add 10.17.14.219/24 dev ens192

# 测试完成后，删除IP方法如下：
ip addr del 10.17.14.210/24 dev ens192
```

测试
```
# 订阅端执行
./emqtt_bench sub -t "\$share/group1/device/+/telemetry" -c 5 -h emqx-node1.cluster.local -u "test" -P "password@123"

# 10个发布端分别执行
./emqtt_bench pub --ifaddr 10.17.14.210 -t "device/%i/telemetry" -c 50000 -R 500 -I 60000 -h emqx-node1.cluster.local -s 1024 -m "$(base64 /dev/urandom | head -c 1024)" -u "test" -P "password@123"
./emqtt_bench pub --ifaddr 10.17.14.211 -t "device/%i/telemetry" -c 50000 -R 500 -I 60000 -h emqx-node1.cluster.local -s 1024 -m "$(base64 /dev/urandom | head -c 1024)" -u "test" -P "password@123"
./emqtt_bench pub --ifaddr 10.17.14.212 -t "device/%i/telemetry" -c 50000 -R 500 -I 60000 -h emqx-node1.cluster.local -s 1024 -m "$(base64 /dev/urandom | head -c 1024)" -u "test" -P "password@123"
./emqtt_bench pub --ifaddr 10.17.14.213 -t "device/%i/telemetry" -c 50000 -R 500 -I 60000 -h emqx-node1.cluster.local -s 1024 -m "$(base64 /dev/urandom | head -c 1024)" -u "test" -P "password@123"
./emqtt_bench pub --ifaddr 10.17.14.214 -t "device/%i/telemetry" -c 50000 -R 500 -I 60000 -h emqx-node1.cluster.local -s 1024 -m "$(base64 /dev/urandom | head -c 1024)" -u "test" -P "password@123"
./emqtt_bench pub --ifaddr 10.17.14.215 -t "device/%i/telemetry" -c 50000 -R 500 -I 60000 -h emqx-node1.cluster.local -s 1024 -m "$(base64 /dev/urandom | head -c 1024)" -u "test" -P "password@123"
./emqtt_bench pub --ifaddr 10.17.14.216 -t "device/%i/telemetry" -c 50000 -R 500 -I 60000 -h emqx-node1.cluster.local -s 1024 -m "$(base64 /dev/urandom | head -c 1024)" -u "test" -P "password@123"
./emqtt_bench pub --ifaddr 10.17.14.217 -t "device/%i/telemetry" -c 50000 -R 500 -I 60000 -h emqx-node1.cluster.local -s 1024 -m "$(base64 /dev/urandom | head -c 1024)" -u "test" -P "password@123"
./emqtt_bench pub --ifaddr 10.17.14.218 -t "device/%i/telemetry" -c 50000 -R 500 -I 60000 -h emqx-node1.cluster.local -s 1024 -m "$(base64 /dev/urandom | head -c 1024)" -u "test" -P "password@123"
./emqtt_bench pub --ifaddr 10.17.14.219 -t "device/%i/telemetry" -c 50000 -R 500 -I 60000 -h emqx-node1.cluster.local -s 1024 -m "$(base64 /dev/urandom | head -c 1024)" -u "test" -P "password@123"
```

实测单节点最大并发连接数达到25万，每分钟1条消息，最大25万连接，吞吐量 4300msg/s

#### emqtt-bench 测试工具统计
![](image4_1.png)

* 发布速率：4384 msg/s, 订阅速率：4384 msg/s

#### EMQX 官方看板统计数据
![](image4_2.png)

* 连接数：250002，发布速率：4354 msg/s，订阅速率：4354 msg/s

#### CPU、内存使用率统计

**EMQ负载**
![](image4_3.png)

**订阅端负载**
![](image4_4.png)

* EMQ 服务端：948%、6G
* 订阅端：31.7%、612M

## 吞吐量测试

### 用例一：测试2k msg/s 吞吐量 (2000个发布者，每秒发布1条消息，2个订阅者，消息大小：1024字节，qos 0)

```
# 订阅端执行
./emqtt_bench sub -t "\$share/group1/device/+/telemetry" -c 2 -h emqx-node1.cluster.local -u "test" -P "password@123"

# 发布端执行
./emqtt_bench pub -t "device/%i/telemetry" -c 2000 -L 300000 -h emqx-node1.cluster.local -s 1024 -m "$(base64 /dev/urandom | head -c 1024)" -u "test" -P "password@123"
```

#### emqtt-bench 测试工具统计
![](image5_1.png)

* 发布速率：1967 msg/s, 订阅速率：1967 msg/s
* 消息发布数：30万，消息订阅数：30万，消息丢失数：0

#### EMQX 官方看板统计数据
![](image5_2.png)

* 连接数：2002，发布速率：1989 msg/s，订阅速率：1989 msg/s

#### CPU、内存使用率统计

**EMQ负载**
![](image5_3.png)

**订阅端负载**
![](image5_4.png)

* EMQ 服务端：153%、588M
* 订阅端：26%、610M

### 用例二：测试1w msg/s 吞吐量 (10000个发布者，每秒发布1条消息，2个订阅者，消息大小：1024字节，qos 0)

```
# 订阅端执行
./emqtt_bench sub -t "\$share/group1/device/+/telemetry" -c 2 -h emqx-node1.cluster.local -u "test" -P "password@123"

# 发布端执行
./emqtt_bench pub -t "device/%i/telemetry" -R 300 -c 10000 -L 1500000 -h emqx-node1.cluster.local -s 1024 -m "$(base64 /dev/urandom | head -c 1024)" -u "test" -P "password@123"
```

#### emqtt-bench 测试工具统计
![](image6_1.png)

* 发布速率：9998 msg/s, 订阅速率：9998 msg/s
* 消息发布数：150万，消息订阅数：150万，消息丢失数：0

#### EMQX 官方看板统计数据
![](image6_2.png)

* 连接数：10002，发布速率：10010 msg/s，订阅速率：10009 msg/s

#### CPU、内存使用率统计

**EMQ负载**
![](image6_3.png)


**订阅端负载**
![](image6_4.png)

* EMQ 服务端：340%、968M
* 订阅端：53%、610M

### 用例三：测试5w msg/s 吞吐量 （50000个发布者，每秒发布1条消息，5个订阅者，消息大小：1024字节，qos 0）

```
# 订阅端执行
./emqtt_bench sub -t "\$share/group1/device/+/telemetry" -c 2 -h emqx-node1.cluster.local -u "test" -P "password@123"

# 发布端执行
./emqtt_bench pub -t "device/%i/telemetry" -R 500 -c 50000 -L 10000000 -h emqx-node1.cluster.local -s 1024 -m "$(base64 /dev/urandom | head -c 1024)" -u "test" -P "password@123"
```

#### emqtt-bench 测试工具统计
![](image7_1.png)

* 发布速率：50000 msg/s, 订阅速率：50000 msg/s
* 消息发布数：1000万，消息订阅数：1000万，消息丢失数：0

#### EMQX 官方看板统计数据
![](image7_2.png)

* 连接数：50005，发布速率：49999 msg/s，订阅速率：49995 msg/s

#### CPU、内存使用率统计

**EMQ负载**
![](image7_3.png)

**订阅端负载**
![](image7_4.png)

* EMQ 服务端：1679%、3G
* 订阅端：164%、610M

### 用例四： 测试10w msg/s 吞吐量 （5万个发布者，每秒发布2条消息，20个订阅者，消息大小：1024字节，qos 0）

```
# 订阅端执行
./emqtt_bench sub -t "\$share/group1/device/+/telemetry" -c 20 -h emqx-node1.cluster.local -u "test" -P "password@123"

# 发布端执行
./emqtt_bench pub -t "device/%i/telemetry" -R 500 -I 500 -c 50000 -L 15000000 -h emqx-node1.cluster.local -s 1024 -m "$(base64 /dev/urandom | head -c 1024)" -u "test" -P "password@123"
```

#### emqtt-bench 测试工具统计
![](image8_1.png)

* 发布速率：100010 msg/s, 订阅速率：100010 msg/s
* 消息发布数：1500万，消息订阅数：1500万，消息丢失数：0

#### EMQX 官方看板统计数据
![](image8_2.png)

* 连接数：50020，发布速率：100000 msg/s，订阅速率：100006 msg/s

#### CPU、内存使用率统计

**EMQ负载**
![](image8_3.png)

**订阅端负载**
![](image8_4.png)

* EMQ 服务端：1650%、4.3G
* 订阅端：537%、635M

## 测试结果

多对一发布订阅场景，EMQX 5.8.6 单节点可支持25w以上并发连接，测试吞吐量达 10w msg/s 以上 

**并发连接数测试结果**
| 并发连接数 | 发布者数 | 订阅者数 | 发布速率 | 订阅速率 | 消息发布数 | 消息订阅数 | 消息丢失数 | EMQ服务端 CPU | EMQ服务端内存 |
| :-- | :-- | :-- | :-- | :-- | :-- | :-- | :-- | :-- | :-- |
| 2000 | 2000 | 2 | 65 msg/s | 65 msg/s | 6000 | 6000 | 0 | 10% | 343 M |
| 5万 | 5万 | 2 | 966 msg/s | 933 msg/s | 30万 | 30万 | 0 | 220% | 1.3 G |
| 10万 | 10万 | 2 | 1487 msg/s | 1487 msg/s | 60万 | 60万 | 0 | 312% | 2.5 G |
| 25万 | 25万 | 2 | 4354 msg/s | 4354 msg/s | 150万 | 150万 | 0 | 948% | 6G |

**吞吐量测试结果**
| 测试吞吐量 | 发布者数 | 订阅者数 | 发布速率 | 订阅速率 | 消息发布数 | 消息订阅数 | 消息丢失数 | EMQ服务端 CPU | EMQ服务端内存 |
| :-- | :-- | :-- | :-- | :-- | :-- | :-- | :-- | :-- | :-- |
| 2000 msg/s | 2000 | 2 | 1967 msg/s | 1967 msg/s | 30万 | 30万 | 0 | 153% | 588 M |
| 1万 msg/s | 1万 | 2 | 9998 msg/s | 9998 msg/s | 150万 | 150万 | 0 | 340% | 968 M |
| 5万 msg/s | 5万 | 5 | 50000 msg/s | 50000 msg/s | 1000万 | 1000万 | 0 | 1679% | 3 G |
| 10万 msg/s | 10万 | 20 | 100010 msg/s | 100010 msg/s | 1500万 | 1500万 | 0 | 1650% | 4.3 G |

## 参考文档
* [使用 eMQTT-Bench 进行性能测试](https://docs.emqx.com/zh/emqx/latest/performance/benchmark-emqtt-bench.html)
* [性能工具之emqtt-bench BenchMark 测试示例](https://developer.aliyun.com/article/1487698)