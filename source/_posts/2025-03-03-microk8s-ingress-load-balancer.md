---
layout: next
title: Microk8s Ingress实现七层负载均衡
date: 2025-03-03 20:11:00
categories: kubernetes
tags: kubernetes
---

# Microk8s Ingress是什么
Ingress是k8s的一种资源对象，用于管理外部对集群内服务的访问, 它通过提供一个统一的入口点，将外部流量路由到集群内部的不同服务。

# Microk8s Ingress用于解决什么问题
k8s集群中服务默认只能在集群内访问。 如果需要从外部访问服务，通常需要使用NodePort或LoadBalancer类型服务，这两个服务都存在一些问题
* NodePort会占用节点端口，可能导致端口冲突
* LoadBalancer需要云提供商支持, 不适合本地环境
* Ingress提供一种灵活的方式暴露服务，允许通过域名或路径规则将流量路由到不同的服务
# Microk8s Ingress基本原理
* Microk8s内置了一个Nginx的Ingress Controller, 负责监听k8s中的Ingress资源，当检测到Ingress资源更新时, 动态更新Nginx配置文件
* 外部流量先到达Nginx, 再基于域名和URL将请求转发到Service, Service再将流量分发到Pod

<!-- more -->

```
# kubectl -n ingress get pod nginx-ingress-microk8s-controller-nrftt
NAME                                      READY   STATUS    RESTARTS   AGE
nginx-ingress-microk8s-controller-nrftt   1/1     Running   0          15m

# ps axf 
 860646 ?        Sl     0:00 /var/lib/snapd/snap/microk8s/7665/bin/containerd-shim-runc-v2 -namespace k8s.io -id 8fb71f54235bf26664240c8b9272b623a70d
 860668 ?        Ss     0:00  \_ /pause
 860700 ?        Ss     0:00  \_ /usr/bin/dumb-init -- /nginx-ingress-controller --configmap=ingress/nginx-load-balancer-microk8s-conf --tcp-services-configmap=ingress/nginx-ingress-tcp-microk8s-conf --udp-services-configmap=ingress/nginx-ingress-udp-microk8s-conf --ingress-class=public   --publish-status-address=127.0.0.1 --default-ssl-certificate=default/example-tls
 860712 ?        Ssl    0:00      \_ /nginx-ingress-controller --configmap=ingress/nginx-load-balancer-microk8s-conf --tcp-services-configmap=ingress/nginx-ingress-tcp-microk8s-conf --udp-services-configmap=ingress/nginx-ingress-udp-microk8s-conf --ingress-class=public   --publish-status-address=127.0.0.1 --default-ssl-certificate=default/example-tls
 860782 ?        S      0:00          \_ nginx: master process /usr/bin/nginx -c /etc/nginx/nginx.conf
 860786 ?        Sl     0:00              \_ nginx: worker process
 860787 ?        Sl     0:00              \_ nginx: worker process
 860788 ?        Sl     0:00              \_ nginx: worker process
 860789 ?        Sl     0:00              \_ nginx: worker process
 860790 ?        S      0:00              \_ nginx: cache manager process
```

# 实践: 使用Microk8s Ingress配置HTTP/TCP负载均衡
首先安装并启动Microk8s和Ingress插件, 参考: [在RockyLinux9.4上安装Microk8s](https://blog.csdn.net/pcj_888/article/details/144169716)
演示案例如下图:
![](image1.png)
## 支持HTTP负载均衡

用Flask写两个简单的HTTP服务

app.py
```
from flask import Flask

app = Flask(__name__)

@app.route('/foo/')
def hello_world():
    return "Hello, foo!"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

Dockerfile
```
FROM python:3.9-slim
WORKDIR /app
COPY . /app
RUN pip install flask
EXPOSE 8080
CMD ["python", "app.py"]
```

构建docker镜像
```
docker build -t foo:1.0 -f Dockerfile .
docker save foo:1.0 > foo.tar
```

创建k8s资源, 创建Deployment
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: foo-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: foo
  template:
    metadata:
      labels:
        app: foo
    spec:
      containers:
      - name: foo
        image: foo:1.0
        ports:
        - containerPort: 8080
```

创建Service
```
apiVersion: v1
kind: Service
metadata:
  name: foo-service
spec:
  selector:
    app: foo
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
```

在Microk8s环境上导入镜像, 创建yaml, 先在集群内部测试一下服务OK
```
microk8s.ctr image *.tar
kubectl create -f *.yaml

# kubectl get svc
default       bar-service   ClusterIP   10.152.183.208   <none>        80/TCP                   11s
default       foo-service   ClusterIP   10.152.183.189   <none>        80/TCP                   25m
# curl 10.152.183.189:80/foo/
Hello Foo!
# curl 10.152.183.208/bar/
Hello Bar!
```

**配置Ingress, 支持集群外访问**
默认情况下, Ingress Controller是通过NodePort类型暴露的, 这里我们改成监听宿主机的80端口, 通过修改Ingress的DaemonSet实现:
创建nginx-ingress-microk8s-controller-patch.yaml
```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ingress-microk8s-controller
  namespace: ingress
spec:
  template:
    spec:
      hostNetwork: true
      containers:
      - name: nginx-ingress-microk8s
        args:
        # must list all others here?
        - /nginx-ingress-controller
        - --configmap=$(POD_NAMESPACE)/nginx-load-balancer-microk8s-conf
        - --tcp-services-configmap=$(POD_NAMESPACE)/nginx-ingress-tcp-microk8s-conf
        - --udp-services-configmap=$(POD_NAMESPACE)/nginx-ingress-udp-microk8s-conf
        - --ingress-class=public
        - ' '
        - --publish-status-address=127.0.0.1
```
修改Ingress的DaemonSet
```
kubectl -n ingress patch ds nginx-ingress-microk8s-controller --patch-file nginx-ingress-microk8s-controller-patch.yaml
```
确认宿主机80端口已经LISTEN
```
ss -antp | grep ":*80"
LISTEN    0      4096                  0.0.0.0:80                   0.0.0.0:*     users:(("nginx",pid=741444,fd=17),("nginx",pid=741437,fd=17))
```

创建Ingress资源, 支持从集群外访问两个HTTP服务
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
  - host: foo.example.com
    http:
      paths:
      - path: /foo/
        pathType: Prefix
        backend:
          service:
            name: foo-service
            port:
              number: 80
  - host: bar.example.com
    http:
      paths:
      - path: /bar/
        pathType: Prefix
        backend:
          service:
            name: bar-service
            port:
              number: 80
```

测试, 从集群外部访问成功
```
curl foo.example.com/foo/
Hello Foo!
curl bar.example.com/bar/
Hello Bar!
```

## 支持TCP负载均衡
Ingress同时支持四层(TCP,UDP)的负载均衡, 例如:

创建一个TCP服务, 监听8888端口 (tcp-service.yaml)
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tcp-server
  namespace: service-foo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tcp-server
  template:
    metadata:
      labels:
        app: tcp-server
    spec:
      containers:
      - name: tcp-container
        image: busybox
        command: ["nc", "-lk", "8888"]
        ports:
        - containerPort: 8888
---
apiVersion: v1
kind: Service
metadata:
  name: tcp-service
  namespace: service-foo
spec:
  selector:
    app: tcp-server
  ports:
    - protocol: TCP
      port: 8888
      targetPort: 8888
  type: ClusterIP
```

配置Ingress的ConfigMap, 以支持TCP的暴露

创建nginx-ingress-tcp-microk8s-conf-patch.yaml
```
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-ingress-tcp-microk8s-conf
  namespace: ingress
data:
  "8888": tcp-service:8888
```
修改Ingress的ConfigMap
```
kubectl -n ingress patch cm nginx-ingress-tcp-microk8s-conf --patch-file nginx-ingress-tcp-microk8s-conf-patch.yaml
```

从集群外部测试, 访问成功
```
# telnet foo.example.com 8888
Connected to foo.example.com.
Escape character is '^]'
```

## Ingress 支持HTTPS

生成证书和私钥
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=example.com/O=Example Company"
```
创建kubernetes secret
```
microk8s kubectl create secret tls example-tls --cert=tls.crt --key=tls.key -n default
```
改一下Ingress的DaemonSet, 添加一行`--default-ssl-certificate`
```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ingress-microk8s-controller
  namespace: ingress
spec:
  template:
    spec:
      hostNetwork: true
      containers:
      - name: nginx-ingress-microk8s
        args:
        # must list all others here?
        - /nginx-ingress-controller
        - --configmap=$(POD_NAMESPACE)/nginx-load-balancer-microk8s-conf
        - --tcp-services-configmap=$(POD_NAMESPACE)/nginx-ingress-tcp-microk8s-conf
        - --udp-services-configmap=$(POD_NAMESPACE)/nginx-ingress-udp-microk8s-conf
        - --ingress-class=public
        - ' '
        - --publish-status-address=127.0.0.1
        - --default-ssl-certificate=default/example-tls
```
```
kubectl -n ingress patch ds nginx-ingress-microk8s-controller --patch-file nginx-ingress-microk8s-controller-patch.yaml
```

从集群外部测试, HTTP和HTTPS都可以访问
```
# curl foo.example.com/foo/
Hello Foo!
# curl https://bar.example.com/bar/ -k
Hello Bar!
```

## 问题
Q: 可能遇到导入Microk8s镜像失败, `microk8s.ctr image list`查询镜像类型为text/html, 只有几百k
A: 原因一般是之前导入镜像前就创建了pod, Microk8s尝试从网络获取镜像失败; 解决方法是先`microk8s.ctr image rm`删除旧的镜像, 再重新导入即可

## 参考
【1】 [https://microk8s.io/docs/addon-ingress](https://microk8s.io/docs/addon-ingress)
【2】 [https://kubernetes.io/docs/concepts/services-networking/ingress/](https://kubernetes.io/docs/concepts/services-networking/ingress/)