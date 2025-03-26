---
layout: next
title: 使用 ChartMuseum 容器搭建私有 Helm Chart 仓库
date: 2025-03-23 16:50:15
categories: Kubernetes
tags: Kubernetes
---



# 前言
本文介绍如何在 Rocky Linux 9.5 上使用 ChartMuseum 搭建一个私有的 Helm Chart 仓库，并启用 HTTPS 和 Basic 认证以提高安全性

# 环境准备
* 一台Rocky Linux 9.5 x86_64, 作为Helm Repo, 安装Docker, 域名: `myhelmrepo.com`
* Kubernetes 集群 v1.28.x, Helm v3.16.0

# 1. 安装并启动 ChartMuseum
(1) 启动 ChartMuseum 容器
```
docker run -d \
  --name private-helm-repo \
  -p 8080:8080 \
  --restart=always \
  -e STORAGE=local \
  -e STORAGE_LOCAL_ROOTDIR=/charts \
  -v /charts:/charts \
  --user root \
  swr.cn-north-4.myhuaweicloud.com/ddn-k8s/ghcr.io/helm/chartmuseum:v0.16.2
```
<!-- more -->
(2) 验证 ChartMuseum 是否正常运行
```
curl http://myhelmrepo.com:8080/index.yaml
```

如果返回类似以下内容，则说明服务已成功启动：
```
apiVersion: v1
entries: {}
generated: "2025-03-23T07:32:46Z"
serverInfo: {}
```

# 2. 上传 Helm Chart 到仓库
(1) 上传 Chart
假设你有一个名为 [flask-app-chart-0.1.0.tgz](https://github.com/PCJ600/helm-chart-demo/blob/main/charts/flask-app-chart-0.1.0.tgz) 的 Helm Chart，可以通过以下命令将其上传到仓库：
```
curl --data-binary "@flask-app-chart-0.1.0.tgz" http://myhelmrepo.com:8080/api/charts
```
如果返回以下内容，则表示上传成功：
```
{"saved":true}
```
(2) 验证上传结果
再次检查 index.yaml 文件以确认 Chart 成功添加
```
curl http://myhelmrepo.com:8080/index.yaml
apiVersion: v1
entries:
  flask-app-chart:
  - apiVersion: v2
    appVersion: "1.0"
    created: "2025-03-23T07:38:06.252423409Z"
    description: A Helm chart for deploying a simple Flask app
    digest: c6ad11cf3c0b9c2068b5cb3502c8faabc700646b4a7fe9d1c3e779827658a474
    name: flask-app-chart
    type: application
    urls:
    - charts/flask-app-chart-0.1.0.tgz
    version: 0.1.0
generated: "2025-03-23T07:38:06Z"
serverInfo: {}
```

# 3. 添加私有 Helm 仓库
(1) 添加私有仓库
```
helm repo add my-private-repo http://myhelmrepo.com:8080
```
更新仓库缓存：
```
helm repo update
```
如果返回以下内容，则表示仓库添加成功：
```
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "my-private-repo" chart repository
Update Complete. ⎈Happy Helming!⎈
```
(2) 查看仓库列表
```
helm repo list

NAME            URL
my-private-repo http://myhelmrepo.com:8080
```
(3) 查看仓库中的 Charts
```
helm search repo -l flask

NAME                            CHART VERSION   APP VERSION     DESCRIPTION
my-private-repo/flask-app-chart 0.1.0           1.0             A Helm chart for deploying a simple Flask app
```

# 4. 下载或安装 Chart
(1) 下载 Chart
下载刚才上传的 Chart：
```
helm pull my-private-repo/flask-app-chart
```
解压并查看内容：
```
tar -tvf flask-app-chart-0.1.0.tgz
```
(2) 安装 Chart
通过以下命令安装 Chart：
```
helm install flask-app my-private-repo/flask-app-chart --namespace flask-app
```
参数说明：
* flask-app: 指定本次安装的 Release 名称。
* my-private-repo/flask-app-chart: 指定 Chart 的完整路径。
* --namespace flask-app: 指定安装的目标命名空间。

# 5. 启用 HTTPS 和 Basic 认证
为了提高安全性，在生产环境中需要启用 HTTPS 和 Basic 认证。

(1) 安装相关软件
```
yum install nginx httpd-tools -y
```
(2) 启用 Basic 认证
重新启动 ChartMuseum 容器并启用认证：
```
docker run -d \
  --name private-helm-repo \
  -p 127.0.0.1:8080:8080 \
  --restart=always \
  -e STORAGE=local \
  -e STORAGE_LOCAL_ROOTDIR=/charts \
  -v /charts:/charts \
  -e AUTH_ANONYMOUS_GET=false \
  -e BASIC_AUTH_USER=helmuser \
  -e BASIC_AUTH_PASS=123456 \
  --user root \
  swr.cn-north-4.myhuaweicloud.com/ddn-k8s/ghcr.io/helm/chartmuseum:v0.16.2
```
测试认证是否生效：
```
# 不带认证访问
curl localhost:8080
# 返回 {"error":"unauthorized"}

# 带认证访问
curl -u helmuser:123456 localhost:8080
# 返回 200 OK
```
(3) 启用 HTTPS
生成自签名证书, 创建目录并生成自签名 SANs 证书：
```
sudo mkdir -p /etc/nginx/ssl/myhelmrepo.com
cd /etc/nginx/ssl/myhelmrepo.com
```
创建 OpenSSL 配置文件（openssl.cnf）：
```
[ req ]
default_bits       = 2048
distinguished_name = req_distinguished_name
req_extensions     = req_ext
prompt             = no

[ req_distinguished_name ]
C  = CN
ST = State
L  = City
O  = Organization
OU = Organizational Unit
CN = myhelmrepo.com

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = myhelmrepo.com
DNS.2 = localhost
IP.1 = 127.0.0.1
```
生成证书和密钥：
```
openssl req -x509 -config openssl.cnf -extensions 'req_ext' -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/myhelmrepo.com/myhelmrepo.com.key \
  -out /etc/nginx/ssl/myhelmrepo.com/myhelmrepo.com.crt
```
编辑 Nginx 主配置文件(/etc/nginx/nginx.conf)：
```
server {
    listen 443 ssl;
    server_name myhelmrepo.com;

    ssl_certificate /etc/nginx/ssl/myhelmrepo.com/myhelmrepo.com.crt;
    ssl_certificate_key /etc/nginx/ssl/myhelmrepo.com/myhelmrepo.com.key;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
重启 Nginx：
```
systemctl restart nginx
```
测试 HTTPS
```
curl https://myhelmrepo.com/index.yaml -k
# 返回 401 Authorization Required

curl -u helmuser:123456 https://myhelmrepo.com/index.yaml -k
# 返回 200 OK
```
(4) 配置 Helm 使用 HTTPS
由于使用的是自签名证书，Helm默认无法信任。将证书复制到本地并配置 Helm
```
mkdir -p ~/.helm/certs
scp root@myhelmrepo.com:/etc/nginx/ssl/myhelmrepo.com/myhelmrepo.com.crt ~/.helm/certs/
```
重新添加仓库：
```
helm repo rm my-private-repo
helm repo add my-private-repo https://myhelmrepo.com \
  --ca-file ~/.helm/certs/myhelmrepo.com.crt \
  --username helmuser \
  --password 123456
```
更新仓库缓存并查看Chart
```
helm repo update
helm search repo -l flask
NAME                            CHART VERSION   APP VERSION     DESCRIPTION
my-private-repo/flask-app-chart 0.1.0           1.0             A Helm chart for deploying a simple Flask app
```
安装Chart, 测试OK
```
helm install flask-app my-private-repo/flask-app-chart --namespace flask-app
```

# 参考
【1】[https://chartmuseum.com/docs/](https://chartmuseum.com/docs/)
