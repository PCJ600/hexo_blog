---
layout: next
title: Kubernetes滚动更新实践
date: 2025-01-01 21:56:40
categories: Kubernetes
tags: Kubernetes
---

# 前言
在我之前的项目中，对微服务升级采用的做法是删除整个namespace， 再重新应用所有yaml。 这种方式简单粗暴，但是不可避免的导致服务中断，影响了用户体验
为了解决更新服务导致的服务中断问题， Kubernetes提供了一种有效的服务升级方式 —— 滚动更新(Rolling Update)

# 什么是滚动更新(RollingUpdate)
滚动更新是一种部署策略。 允许用户逐步替换旧的Pod实例为新版本，而不是一次性替换所有Pod，从而实现零停机时间的部署更新。 解决了如下问题:
* 最小化停机时间， 滚动更新可以在不完全停止服务情况下进行，提高用户体验
* 故障恢复， 如果新版本出现问题，可以迅速回滚到之前的稳定版本
* 平滑流量迁移， 避免瞬间全部更新导致的流量冲击和服务中断

RollingUpdate策略有两个重要参数：
* maxUnavailable， 滚动更新时最多可以有多少个Pod不可用。 默认值为25%，这意味着如果有一个包含4个Pod的服务， 更新期间至少有3个Pod可用
* maxSurge， 滚动更新时可以临时超出期望副本数的额外Pod数， 默认值为25%

说明: 对于关键业务， 可以用保守的设置， 比如maxUnavailable=0， maxSurge=1; 较大的maxSurge可以加快更新速度，但增加了集群压力; 根据实际情况调整这两个参数


## 实践案例

写一个简单的微服务， Pod副本数设置为5， 观察Kubernetes滚动更新过程
<!-- more -->

1、首先创建一个简单微服务
创建Flask的app.py
```py
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello v1!"

if __name__ == "__main__":
    app.run(host='0.0.0.0')
```
写一个Dockerfile
```
FROM python:3.9-slim
RUN pip install --no-cache-dir -r requirements.txt
WORKDIR /app
COPY . /app
EXPOSE 5000
ENV FLASK_APP=app.py
CMD ["flask"， "run"， "--host=0.0.0.0"]
```
构建镜像
```
docker build -t flask-app:1.0 .
docker save flask-app:1.0 > flask-app-1_0.tar
```

2、 微服务部署到k8s
创建一个Deployment
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
spec:
  replicas: 5
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
      - name: flask-app
        image: docker.io/library/flask-app:1.0
        ports:
        - containerPort: 5000
```
创建Service
```
apiVersion: v1
kind: Service
metadata:
  name: flask-service
spec:
  type: NodePort
  selector:
    app: flask-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
```

3、 执行滚动更新
改一下第一步的app.py， 构建第二个镜像， 用于测试滚动更新
```
docker build -t flask-app:2.0 .
docker save flask-app:2.0 > flask-app-2_0.tar
```
执行滚动更新
```
kubectl set image deployment/flask-app flask-app=docker.io/library/flask-app:2.0
```

4、 观察滚动更新过程
```
kubectl rollout status deployment/flask-app
```
同时， 用watch查看滚动更新过程， 可以看出Pod是一边创建一边删除的
```
watch kubectl get pods

Every 2.0s: kubectl get pods                                                                                                                                                                                                         localhost.localdomain: Sat Mar  8 18:43:27 2025
NAME                         READY   STATUS        RESTARTS   AGE
flask-app-585869cc8f-27csx   1/1     Running       0          6s
flask-app-585869cc8f-46jps   1/1     Running       0          3s
flask-app-585869cc8f-bnbg7   1/1     Running       0          5s
flask-app-585869cc8f-c9gwx   1/1     Running       0          6s
flask-app-585869cc8f-klwrn   1/1     Running       0          4s
flask-app-6bc5857dc8-2r2l6   1/1     Terminating   0          63s
flask-app-6bc5857dc8-9q4vc   1/1     Terminating   0          63s
flask-app-6bc5857dc8-9vjds   1/1     Terminating   0          61s
flask-app-6bc5857dc8-mxqjg   1/1     Terminating   0          61s
flask-app-6bc5857dc8-wvrbl   1/1     Terminating   0          63s
```

5、 滚动回滚
```
kubectl rollout undo deployment/flask-app
```

注: 除了改变镜像， 如果对deployment其他部分做了修改， 同样可以触发滚动更新。 通过kubectl apply更新Deployment， 再`watch kubectl get pods`观察滚动更新过程

# 参考
[https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/)
