---
layout: next
title: 使用Helm安装、 升级、 回滚Kubernetes应用
date: 2025-03-23 21:05:00
top: 95
categories: Kubernetes
tags: Kubernetes
---


# 前言
在我之前做的项目里，我们对Microk8s微服务的更新是通过自制tar包的方式做的, tar包存储了镜像和YAML文件。 每次升级时，我们需要先删除所有的YAML资源，然后重新创建新的资源。 这种方式存在以下问题：
* 服务中断：由于需要先删除旧资源再创建新资源，这会导致短暂的服务中断，影响用户体验
* 复杂的回滚逻辑：如果升级失败，回滚到之前的版本变得非常复杂，需要手动恢复旧的YAML文件，并且容易出错

了解到Helm可以有效解决以上问题, Helm是Kubernetes 的包管理工具，方便用户快速发现、 共享和使用Kubernetes构建的应用。 以下举例演示如何使用Helm实现安装、升级、回滚操作

<!-- more -->

# 环境准备
* Docker私有仓库 (https://mydockerregistry.com:5000), 参考：[搭建Docker私有仓库](https://blog.csdn.net/pcj_888/article/details/146445826)
* Helm私有仓库 (https://myhelmrepo.com)  参考：[搭建私有 Helm Chart 仓库](https://blog.csdn.net/pcj_888/article/details/146458002)
* Kubernetes集群 v1.28.x, Helm版本 v3.16.0 参考: [kubeadm+keepalived+HAproxy搭建高可用kubernetes集群](https://blog.csdn.net/pcj_888/article/details/144240636)
* 为了使外部流量能访问集群内的服务，需要安装一个Ingress控制器, 参考： [Helm快速上手，使用Helm安装nginx-ingress](https://blog.csdn.net/pcj_888/article/details/146460834)


# 创建一个Flask应用
创建一个简单的 Flask 应用，并准备两个版本：
* 1.0版本, 输出 "Hello, foo!"
* 2.0版本, 输出 "Hello, foo v2!"

文件结构
```
flask-app/
├── app.py
├── requirements.txt
└── Dockerfile
```

app.py
```py
from flask import Flask

app = Flask(__name__)

@app.route('/', methods=['GET'])
def root():
    return "hello foo"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
```

requirements.txt
```
Flask
```

Dockerfile
```
FROM rockylinux:9.3

RUN dnf install -y python3 python3-pip

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY app.py .

CMD ["python3", "app.py"]
```

## 构建并测试镜像
构建镜像
```
docker build -t mydockerregistry.com:5000/flask-app:1.0 .
```
本地运行容器测试
```
docker run -d -p 8000:8000 mydockerregistry.com:5000/flask-app:1.0
curl localhost:8000/
hello foo
```
测试OK后, 推送镜像到repo
```
docker push mydockerregistry.com:5000/flask-app:1.0
```

# 配置Kubernetes环境

## 创建secret存储私有镜像仓库认证信息
为了从私有镜像仓库拉取镜像，我们需要在Kubernetes集群中创建一个Secret，用于存储认证信息。 这一步是必要的，因为Kubernetes默认无法直接访问需要认证的私有镜像仓库
执行以下命令创建Secret
```
kubectl create secret docker-registry my-registry-secret \
  --docker-server=mydockerregistry.com:5000 \
  --docker-username=dockeruser \
  --docker-password=123456 \
  --namespace flask-app
```
说明：
* 该 Secret 将被 Helm Chart 中的 imagePullSecrets 引用，确保 Kubernetes 能够从私有镜像仓库拉取镜像。

## 配置 Containerd 忽略证书验证（仅限测试环境）
在每个节点上，编辑 /etc/containerd/config.toml 文件，添加以下配置以忽略私有镜像仓库的证书验证
```
[plugins."io.containerd.grpc.v1.cri".registry]
  [plugins."io.containerd.grpc.v1.cri".registry.configs]
    [plugins."io.containerd.grpc.v1.cri".registry.configs."mydockerregistry.com:5000".tls]
      insecure_skip_verify = true
```
重启 containerd 服务以应用更改
```
systemctl restart containerd
```

# 创建Helm Chart
Helm Chart 是 Helm 的核心组件，它允许我们以模板化的方式定义 Kubernetes 资源。下面我们创建一个名为 flask-app-chart 的 Chart。
```
helm create flask-app-chart
```
生成的文件结构如下：
```
flask-app-chart/
├── charts
├── Chart.yaml
├── templates
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt
│   ├── serviceaccount.yaml
│   ├── service.yaml
└── values.yaml
```

**修改Chart文件**
逐个修改以下文件, 或者直接使用: [https://github.com/PCJ600/helm-chart-demo/tree/main/charts/v1.0/flask-app-chart](https://github.com/PCJ600/helm-chart-demo/tree/main/charts/v1.0/flask-app-chart)

Chart.yaml
```
apiVersion: v2
name: flask-app-chart
description: A Helm chart for deploying a simple Flask app
type: application
version: 1.0.0 # Chart 版本
appVersion: "1.0" # 应用版本
```

values.yaml
```
# 全局名称覆盖
fullnameOverride: ""
nameOverride: ""

# 副本数
replicaCount: 6

# 镜像配置
image:
  repository: "mydockerregistry.com:5000/flask-app"
  tag: "1.0"
  pullPolicy: "IfNotPresent"
  pullSecrets:
    - my-registry-secret

# 资源限制
resources:
  limits:
    cpu: "500m"
    memory: "512Mi"
  requests:
    cpu: "250m"
    memory: "256Mi"

# 环境变量
env:
  ENV_VAR_1: "value1"
  ENV_VAR_2: "value2"

# Ingress 配置
ingress:
  enabled: true
  host: "flask-app.example.com"
  path: "/foo(/|$)(.*)"
  pathType: "ImplementationSpecific"

# Service 配置
service:
  port: 8000

serviceAccount:
  create: false
  name: ""

autoscaling:
 enabled: false
```

templates/deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "flask-app-chart.fullname" . }}
  labels:
    app: {{ include "flask-app-chart.name" . }}
    chart: {{ .Chart.Name }}
    release: {{ .Release.Name }}
    heritage: {{ .Release.Service }}
spec:
  replicas: {{ .Values.replicaCount | default 1 }}
  selector:
    matchLabels:
      app: {{ include "flask-app-chart.name" . }}
  template:
    metadata:
      labels:
        app: {{ include "flask-app-chart.name" . }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default "latest" }}"
          imagePullPolicy: {{ .Values.image.pullPolicy | default "IfNotPresent" }}
          ports:
            - containerPort: 8000
              protocol: TCP
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          env:
            {{- if .Values.env }}
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
            {{- end }}
      imagePullSecrets:
        {{- if .Values.image.pullSecrets }}
        {{- range .Values.image.pullSecrets }}
        - name: {{ . }}
        {{- end }}
        {{- else }}
        - name: my-registry-secret
        {{- end }}
```
注: 配置了imagePullSecrets, 支持从私有仓库拉取镜像

template/service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: {{ include "flask-app-chart.fullname" . }}
  namespace: {{ .Release.Namespace }}
spec:
  selector:
    app: {{ include "flask-app-chart.name" . }}
  ports:
  - port: {{ .Values.service.port | default 8000 }}
    targetPort: {{ .Values.service.port | default 8000 }}
  type: ClusterIP
```

template/ingress.yaml
定义ingress模板, 支持集群外部访问
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "flask-app-chart.fullname" . }}-ingress
  namespace: {{ .Release.Namespace }}
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2 # 将 /foo/(.*) 重写为 /$1
spec:
  ingressClassName: nginx # 指定 IngressClass 名称
  rules:
  - host: {{ .Values.ingress.host | quote }}
    http:
      paths:
      - path: {{ .Values.ingress.path | default "/foo(/|$)(.*)" }}
        pathType: {{ .Values.ingress.pathType | default "ImplementationSpecific" }}
        backend:
          service:
            name: {{ include "flask-app-chart.fullname" . }}
            port:
              number: {{ .Values.service.port | default 8000 }}
```

templates/_helpers.tpl
```
{{/*
Create a default fully qualified app name.
We truncate at 63 characters because some Kubernetes name fields are limited to this (by the DNS naming spec).
*/}}
{{- define "flask-app-chart.fullname" -}}
{{- if .Values.fullnameOverride -}}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- printf "%s-%s" .Chart.Name .Release.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end -}}

{{/*
Create a default name for the chart.
*/}}
{{- define "flask-app-chart.name" -}}
{{- default .Chart.Name .Values.nameOverride -}}
{{- end -}}
```

**本地测试部署**
在正式发布 Chart 到仓库之前，可以先在本地进行测试部署
```
kubectl create ns flask-app
helm install flask-app ./flask-app-chart --namespace flask-app
```
验证OK
```
# curl flask-app.example.com/foo
hello foo
```

测试完成后，可以卸载Chart, 后面准备发布Chart到仓库
```
helm -n flask-app uninstall flask-app
```

# 发布Chart到仓库

检查Chart的正确性
```
helm lint flask-app-chart
# helm lint flask-app-chart/
==> Linting flask-app-chart/
[INFO] Chart.yaml: icon is recommended
```
打包Chart
```
# helm package flask-app-chart
Successfully packaged chart and saved it to: /path/to/flask-app-chart-1.0.0.tgz
```
上传 Chart 到私有 Helm 仓库
```
curl -u "helmuser:123456" --data-binary "@flask-app-chart-1.0.0.tgz" https://myhelmrepo.com/api/charts -k
```
如果需要删除旧版本的 Chart，使用以下命令
```
curl -X DELETE "https://myhelmrepo.com/api/charts/flask-app-chart/0.1.0" -u "helmuser:123456"
```

# 通过Helm Repo安装Chart

添加私有 Helm 仓库
```
helm repo add my-private-repo https://myhelmrepo.com \
  --ca-file ~/.helm/certs/myhelmrepo.com.crt \
  --username helmuser \
  --password 123456
```

更新仓库缓存并查看可用的 Charts
```
helm repo update
helm search repo -l flask
NAME                            CHART VERSION   APP VERSION     DESCRIPTION
my-private-repo/flask-app-chart 1.0.0           1.0             A Helm chart for deploying a simple Flask app
```

安装Chart到指定命名空间flask-app
```
helm install flask-app my-private-repo/flask-app-chart --namespace flask-app --version 1.0.0
```
验证安装是否成功
```
# curl flask-app.example.com/foo
hello foo
```

# 再发布2.0版本的Chart, 用于测试升级和回滚功能

改下app.py, 返回'hello foo v2'
```py
from flask import Flask

app = Flask(__name__)

@app.route('/', methods=['GET'])
def root():
    return "hello foo v2"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
```

构建并推送新版本镜像
```
docker build -t mydockerregistry.com:5000/flask-app:2.0 .
docker push mydockerregistry.com:5000/flask-app:2.0
```

更新Chart配置, 修改Chart.yaml和values.yaml

Chart.yaml
更新版本号为2.0.0, 并设置appVersion为2.0
```
# cat Chart.yaml
apiVersion: v2
name: flask-app-chart
description: A Helm chart for deploying a simple Flask app
type: application
version: 2.0.0
appVersion: "2.0"
```

改下values.yaml, 将镜像版本从1.0更新为2.0
```
image:
  repository: "mydockerregistry.com:5000/flask-app"
  tag: "2.0"
```

打包并上传新的Chart
```
helm package flask-app-chart
curl -u "helmuser:123456" --data-binary "@flask-app-chart-2.0.0.tgz" https://myhelmrepo.com/api/charts -k
```

# 使用Helm升级到新版本
使用`helm upgrade`将应用从1.0版本升级到2.0版本
```
helm upgrade flask-app my-private-repo/flask-app-chart --version 2.0.0 --namespace flask-app --atomic
```
注: Helm支持通过--atomic参数实现原子性操作。如果升级失败，Helm会自动回滚到之前的版本

验证升级是否成功
```
# curl flask-app.example.com/foo
hello foo v2
```

watch观察升级过程, 可以看出是滚动更新方式
```
watch "kubectl get pods -n flask-app"
NAMESPACE          NAME                                         READY   STATUS        RESTARTS         AGE
flask-app          flask-app-chart-flask-app-5c86b6ffdf-4jlqb   1/1     Running       0                31s
flask-app          flask-app-chart-flask-app-5c86b6ffdf-kkw57   1/1     Running       0                30s
flask-app          flask-app-chart-flask-app-5c86b6ffdf-qlnhv   1/1     Running       0                30s
flask-app          flask-app-chart-flask-app-5c86b6ffdf-s9tr5   1/1     Running       0                31s
flask-app          flask-app-chart-flask-app-5c86b6ffdf-w6776   1/1     Running       0                30s
flask-app          flask-app-chart-flask-app-5c86b6ffdf-xwgtp   1/1     Running       0                31s
flask-app          flask-app-chart-flask-app-5d44b77c54-72mhg   0/1     Terminating   0                3m17s
flask-app          flask-app-chart-flask-app-5d44b77c54-gvgv7   0/1     Terminating   0                3m17s
flask-app          flask-app-chart-flask-app-5d44b77c54-mrj5h   0/1     Terminating   0                3m17s
flask-app          flask-app-chart-flask-app-5d44b77c54-r2bdp   1/1     Terminating   0                3m17s
flask-app          flask-app-chart-flask-app-5d44b77c54-w7v67   1/1     Terminating   0                3m17s
```

# 使用Helm回滚到旧版本

查看历史记录
```
helm history flask-app -n flask-app
REVISION        UPDATED                         STATUS          CHART                   APP VERSION     DESCRIPTION
1               Sun Mar 23 19:15:11 2025        superseded      flask-app-chart-1.0.0   1.0             Install complete
2               Sun Mar 23 19:17:57 2025        deployed        flask-app-chart-2.0.0   2.0             Upgrade complete
```
* REVISION 1: 最初安装的1.0的版本
* REVISION 2: 升级到2.0的版本

执行回滚命令, 回滚到1.0版本
```
helm rollback flask-app 1 -n flask-app
Rollback was a success! Happy Helming!
```

验证回滚成功
```
# helm list -n flask-app
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
flask-app       flask-app       3               2025-03-23 19:20:54.762690061 +0800 CST deployed        flask-app-chart-1.0.0   1.0
# curl flask-app.example.com/foo
hello foo
```

# 总结

Helm vs 传统方式的对比
| 特性           | 传统方式                                                                 | Helm                                                                 |
|----------------|--------------------------------------------------------------------------|----------------------------------------------------------------------|
| **版本管理**   | 手动维护多个 YAML 文件，容易混乱。                                       | 版本化管理，清晰记录每个版本的变化。                                 |
| **升级操作**   | 需要手动删除旧版本并应用新版本，可能导致服务中断。                       | 智能化更新，仅更改必要的部分，避免服务中断。                         |
| **回滚操作**   | 需要手动恢复旧版本的 YAML 文件，操作复杂且容易出错。                     | 一键回滚到任意历史版本，简单高效。                                   |
| **差异追踪**   | 难以知道两个版本之间的具体差异，容易遗漏更改。                           | Helm 自动计算差异，确保所有更改都被正确应用。                        |
| **配置灵活性** | 需要手动编辑多个 YAML 文件，难以适应不同环境（如 dev、staging、prod）。 | 支持参数化配置，通过 `values.yaml` 动态调整，适应不同环境需求。     |
| **依赖管理**   | 需要手动管理依赖关系，容易遗漏或冲突。                                   | 自动管理依赖关系，确保所有组件协同工作。                             |

# 参考
[https://helm.sh/docs/topics/charts/](https://helm.sh/docs/topics/charts/)
