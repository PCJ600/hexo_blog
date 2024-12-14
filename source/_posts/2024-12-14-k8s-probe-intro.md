---
layout: next
title: kubernetes的三种探针ReadinessProbe、LivenessProbe和StartupProbe,以及使用示例
date: 2024-12-14 14:39:23
categories: k8s
tags: 
- k8s
- Squid
---

## 前言
k8s中的Pod由容器组成，容器运行的时候可能因为意外情况挂掉。为了保证服务的稳定性，在容器出现问题后能进行重启，k8s提供了3种探针

## k8s的三种探针
为了探测容器状态，k8s提供了两个探针: LivenessProbe和ReadinessProbe
* LivenessProbe 存活性探针, 如果探测失败, 就根据Pod重启策略，判断是否要重启容器。
* ReadinessProbe 就绪性探针，如果检测失败，将Pod的IP:Port从对应endpoint列表中删除，防止流量转发到不可用Pod上

对于启动非常慢的应用, LivenessProbe和ReadinessProbe可能会一直检测失败，这会导致容器不停地重启。 所以k8s设计了第三种探针StartupProbe解决这个问题
* StartupProbe探针会阻塞LivenessProbe和ReadinessProbe, 直到满足StartupProbe(Pod完成启动)，再启用LivenessProbe和ReadinessProbe

<!-- more -->

## LivenessProbe和ReadinessProbe支持三种探测方法
* ExecAction 容器中执行指定的命令，退出码为0表示探测成功。
* HTTPGetAction 通过HTTP GET请求容器，如果HTTP响应码在【200，400)，认为容器健康。
* TCPSocketAction 通过容器的IP地址和端口号执行TCP检查。如果建立TCP链接，则表明容器健康。

可以给探针配置可选字段，用来更精确控制LivenessProbe和ReadinessProbe的行为
* initialDelaySeconds: 容器启动后等待多少秒后探针才开始工作，默认是0秒
* periodSeconds: 执行探测的时间间隔，默认为10秒
* timeoutSeconds: 探针执行检测请求后，等待响应的超时时间，默认为1秒
* failureThreshold: 探测失败的重试次数，重试一定次数后认为失败。
* successThreshold: 探针在失败后，被视为成功的最小连续成功数。默认值是 1。 存活和启动探测的这个值必须是 1。最小值是 1。


## 实例: 添加一个LivenessProbe探针
需求: 给Squid Pod添加一个livenessProbe, 每隔10秒检测一次Squid进程启动状态，如果连续3次检测进程异常, 就重启Pod

首先创建一个Squid Pod, 参考: [【k8s实践】 部署Squid](https://pcj600.github.io/2024/1208145435.html)

### 添加一个ExecAction类型的livenessProbe探针
可以通过`squid -k check`命令检测Squid进程运行状态，返回0说明正常, 返回非0值说明进程异常。 
实例: 定义一个ExecAction类型的livenessProbe, deployment配置如下:
```yaml
containers:
  - name: squid
    livenessProbe:
      exec:
        command: ["squid","-k","check"]
          initialDelaySeconds: 5
          periodSeconds: 10
          failureThreshold: 3
```

完整的squid-deployment.yaml如下:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: squid
  namespace: squid
  labels:
    name: squid
spec:
  replicas: 1
  selector:
    matchLabels:
      app: squid
  template:
    metadata:
      labels:
        app: squid
    spec:
      volumes:
        - name: squid-volume
          persistentVolumeClaim:
            claimName: squid-claim
      dnsPolicy: ClusterFirstWithHostNet
      hostNetwork: true
      containers:
        - name: squid
          image: squid:2.0
          imagePullPolicy: IfNotPresent
          livenessProbe: # 添加一个ExecAction类型的探针
            exec:
              command: ["squid","-k","check"] # 如果Squid进程运行, `squid -k check`返回0; 否则返回非0值
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 3
          resources:
            limits:
              memory: "4Gi"
          volumeMounts:
            - mountPath: /var/log/squid
              name: squid-volume
```
重新部署deployment, 通过`kubectl -n squid get deploy squid -o yaml`, 确认探针已配置到deployment.

**测试**
`kubectl exec`进入容器, 使用`squid -k shutdown`手动停止Squid。 等30秒左右，可以观察到容器自动退出重启, 并且从日志中能看到健康检测失败结果:
```
[root@k8s-master ~]# kubectl -n squid exec -it squid-8674587b79-29mq8 -- /bin/bash
[root@k8s-master /]# squid -k shutdown
...
[root@k8s-master /]# squid -k check
2024/12/14 04:53:31| FATAL: failed to open /run/squid.pid: (2) No such file or directory
    exception location: File.cc(190) open
[root@k8s-master /]# echo $?
1

# 等待半分钟左右, 容器自动退出
[root@k8s-master /]# command terminated with exit code 137
[root@k8s-master ~]#

# 此时宿主机上查看Pod，发现RESTARTS次数变为1
[root@k8s-master /]# kubectl -n squid get pods  
squid         squid-8674587b79-29mq8                     1/1     Running   1 (45s ago)     11m

# 通过`kubectl describe`，可以查到探针检测失败的LOG
 kubectl -n squid describe pod squid-8674587b79-29mq8 | grep Unhealthy
  Warning  Unhealthy         2m49s  kubelet            Liveness probe failed: 2024/12/14 04:53:19| FATAL: failed to open /run/squid.pid: (2) No such file or directory
  Warning  Unhealthy  2m39s  kubelet  Liveness probe failed: 2024/12/14 04:53:29| FATAL: failed to open /run/squid.pid: (2) No such file or directory
  Warning  Unhealthy  2m29s  kubelet  Liveness probe failed: 2024/12/14 04:53:39| FATAL: failed to open /run/squid.pid: (2) No such file or directory
```

### 添加一个HttpGet类型的livenessProbe探针
HttpGet探针要求Pod里有一个HTTP的server。 例如：请求`http://localhost:5000/healthz` 探测Pod健康状态，deployment设置如下
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 5000
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3
```
在Pod里写一个HTTP的server, 我这里用的Flask
```py
#!/usr/bin/env python3

# logging
import logging
logger = logging.getLogger(__name__)
file_handler = logging.FileHandler('/var/log/squid/agent.log')
formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')
file_handler.setFormatter(formatter)
logger.addHandler(file_handler)

import subprocess
import traceback

# Flask
from flask import Flask, jsonify, request
app = Flask(__name__)

@app.route('/healthz', methods=['GET'])
def health_check():
    retCode = 0
    try:
        result = subprocess.run(['/usr/sbin/squid', '-k', 'check'])
        retCode = result.returncode
        if retCode == 0:
            return (jsonify({'code': 0, 'msg': 'OK'}), 200)
    except:
        logger.error("health_check %r", traceback.format_exc())
        return (jsonify({'code': -1, 'msg': 'Fail', 'msg': 'Service Unavailable'}), 500)

    return (jsonify({'code': retCode, 'msg': 'Fail'}), 400)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```
再把脚本做到镜像里，安装python3和pip依赖(flask,requests),重新打包部署

**测试**
```
curl localhost:5000/healthz
{"code":0,"msg":"OK"}
```


## 参考
[1] [Correct use of k8s probe](https://juejin-cn.translate.goog/post/7018950464964132877?_x_tr_sl=zh-CN&_x_tr_tl=en&_x_tr_hl=en&_x_tr_pto=sc&_x_tr_hist=true)
[2] [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)