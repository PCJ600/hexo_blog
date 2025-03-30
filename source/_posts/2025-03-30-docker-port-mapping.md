---
layout: next
title: Docker 端口映射原理
date: 2025-03-30 12:58:55
categories: Docker
tags: Docker
---

在 Docker 中，默认情况下容器无法直接与外部网络通信。 为了使外部网络能够访问容器内的服务，Docker 提供了端口映射功能，通过将宿主机的端口映射到容器内的端口，外部可以通过宿主机的IP和端口访问容器内的服务

以下通过动手演示, 安装一个Flask容器, 解释端口映射从外部访问容器的原理

# 安装一个Flask容器

## 文件结构
```
.
├── Dockerfile
└── app.py
```

Dockerfile
```
FROM rockylinux:9.3

RUN dnf update -y && \
    dnf install -y python3 python3-pip && \
    dnf clean all

WORKDIR /app

COPY app.py /app/app.py

RUN pip3 install --no-cache-dir flask

EXPOSE 5000

CMD ["python3", "app.py"]
```

app.py
```py
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return "Hello"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

构建镜像
```
docker build -t flask-app:1.0 .
```
启动容器, 进行端口映射
```
docker run -d -p 80:5000 flask-app:1.0
```

从外部访问 (192.168.52.203是我的虚拟机IP)
```
# curl 192.168.52.203
Hello
```

<!-- more -->

# 端口映射原理

当执行`docker run -d -p 80:5000 flask-app:1.0`时, Docker会配置iptables规则来实现端口映射, 流程如下:

## 1. iptables PREROUTING 链处理, 做DNAT转换

外部请求到达宿主机时, iptables的PREROUTING链处理该请求, 根据DNAT规则将目标IP和端口(192.168.52.203:5000)替换为容器的IP和端口(172.17.0.2:5000)
```
# iptables -t nat -L PREROUTING
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  anywhere             anywhere             ADDRTYPE match dst-type LOCAL

# iptables -t nat -L DOCKER
Chain DOCKER (2 references)
target     prot opt source               destination
RETURN     all  --  anywhere             anywhere
RETURN     all  --  anywhere             anywhere
DNAT       tcp  --  anywhere             anywhere             tcp dpt:http to:172.17.0.2:5000
```

## 2. DNAT转换后的数据包转发到容器

经过DNAT转换后的数据包会发送到虚拟网桥docker0, 再通过veth设备转发到容器的虚拟网络接口(eth0) 
```
宿主机的路由表
# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.52.2    0.0.0.0         UG    100    0        0 ens160
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.52.0    0.0.0.0         255.255.255.0   U     100    0        0 ens160
```

## 3. 容器接受请求并返回响应, 响应报文通过iptables POSTROUTING链处理, 做SNAT转换

容器接受请求并返回响应，响应报文先发送到 docker0, 再通过 SNAT（MASQUERADE）规则，将源IP和端口(172.17.0.2:5000)替换为宿主机的IP和端口，最后发送回客户端
```
# iptables -t nat -L POSTROUTING -v
Chain POSTROUTING (policy ACCEPT 2387 packets, 151K bytes)
 pkts bytes target     prot opt in     out     source               destination
   89  5558 MASQUERADE  all  --  any    !docker0  172.17.0.0/16        anywhere
```

## 附: wireshark抓包结果
* 客户端IP: 192.168.52.202
* 运行容器的宿主机IP: 192.168.52.203
* 容器IP: 172.17.0.2

抓宿主机网卡的包
```
tcpdump -i ens160 -nn tcp and not port 22 -w ens160.pcap
```

![](image1.png)

抓docker0的包
```
tcpdump -i docker0 -nn tcp -w docker0.pcap
```

![](image2.png)


# 参考
[Docker容器访问外部世界 ](https://www.cnblogs.com/heian99/p/12585722.html)