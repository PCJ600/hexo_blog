---
layout: next
title: Helm快速上手, 发布自定义Chart, 搭建私有Helm仓库
date: 2025-03-22 17:10:58
categories: kubernetes
tags: kubernetes
---

# 前言
【交代自己项目]
官方文档: https://helm.sh/

* 什么是Helm, 解决什么问题
* Helm快速上手, 安装官方应用
* 自定义helm chart, 升级, 回滚
* 发布到私有helm仓库

<!-- more -->

# 什么是Helm, Helm解决了哪些问题
Helm是Kubernetes的包管理工具, 方便用户快速发现、共享和使用Kubernetes构建的应用

Helm出现之前, 部署k8s应用存在如下几个问题:
## 1. YAML配置复杂, 部署和管理k8s应用不方便
需要编写大量YAML文件定义k8s资源. 随着项目规模增长, 维护这些配置文件变得困难，尤其是需要对同一组资源进行重复部署的时候(如开发, 测试, 生产环境)
**helm解决方案**
通过Helm模板功能, 开发者可以创建一个Chart, 包含所有必要k8s资源定义, 并使用变量代替硬编码值
例如，在values.yaml中定义默认值，然后在模板中引用这些值。这样，只需修改values.yaml中的几个参数，就可以轻松适应不同的环境或需求，大大减少了出错的可能性并简化了维护工作。

## 2. 更新或回滚到特定版本较为困难
缺乏版本控制功能, 难以更新或回滚到指定版本. 如果使用直接修改线上YAML文件的方式, 可能导致问题, 难以恢复到稳定状态
**helm解决方案**
Helm通过其Release概念支持版本控制, 每次部署或更新应用时, 都创建一个新的Release版本, 如果新版本出现问题, 通过helm rollback可以回滚到之前的稳定版本
此外, helm支持查看每个Release的历史记录, 便于审计和故障排查

## 3. 难以共享或复用k8s配置, 不利于多团队协作
在一个团队内部, 不同项目可能有相似需求, 比如都需要部署同一套系统. 然而, 由于缺少有效的共享机制, 每个项目需要从头开始配置, 浪费了大量时间精力
**helm解决方案**
Helm Charts可以通过仓库(Repository)共享, 每个人可以从仓库中下载官方或其他团队成员发布的Charts, 也可以上传自己的Charts, 实现了k8s配置的共享和复用

# 快速上手Helm

## Helm的三个基本概念(Chart, Repository, Release)
* Chart, 一个Helm安装包，包含了运行一个应用所需要的镜像、依赖和资源定义等
* Release, 在Kubernetes集群上运行的一个Chart实例
* Repository, 发布和存储Chart的仓库

## 安装Helm
首先安装Kubernetes集群, 可以参考: [https://blog.csdn.net/pcj_888/article/details/144240636](https://blog.csdn.net/pcj_888/article/details/144240636)

查看kubernetes版本
```
# kubectl version
Server Version: v1.28.2
```

再查看[官方文档](https://helm.sh/docs/topics/version_skew/), 找到匹配的Helm版本进行安装
```
Helm Version	Supported Kubernetes Versions
3.17.x	1.32.x - 1.29.x
3.16.x	1.31.x - 1.28.x
3.15.x	1.30.x - 1.27.x
3.14.x	1.29.x - 1.26.x
3.13.x	1.28.x - 1.25.x
```
我的Kubernetes版本是1.28.2, 我选择安装Helm 3.16, 方法如下:
```
curl -fsSL -o helm-v3.16.0-linux-amd64.tar.gz https://get.helm.sh/helm-v3.16.0-linux-amd64.tar.gz
tar -zxvf helm-v3.16.0-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
helm version
version.BuildInfo{Version:"v3.16.0", GitCommit:"0d439e1a09683f21a0ab9401eb661401f185b00b", GitTreeState:"clean", GoVersion:"go1.22.6"}
```

## 安装应用
以安装nginx-ingress为例, 首先添加ingress-nginx的官方repo 
```
helm repo add my-ingress https://kubernetes.github.io/ingress-nginx
```
查看repo
```
# helm repo list
NAME            URL
my-ingress      https://kubernetes.github.io/ingress-nginx
```
更新软件源
```
# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "my-ingress" chart repository
Update Complete. ⎈Happy Helming!⎈
```
查看ingress-nginx的版本
```
# helm search repo ingress-nginx
NAME                            CHART VERSION   APP VERSION     DESCRIPTION
my-ingress/ingress-nginx        4.12.0          1.12.0          Ingress controller for Kubernetes using NGINX a...

[root@k8s-master1 ~]# helm search repo ingress-nginx --versions | head -n 4
NAME                            CHART VERSION   APP VERSION     DESCRIPTION
my-ingress/ingress-nginx        4.12.0          1.12.0          Ingress controller for Kubernetes using NGINX a...
my-ingress/ingress-nginx        4.11.4          1.11.4          Ingress controller for Kubernetes using NGINX a...
my-ingress/ingress-nginx        4.11.3          1.11.3          Ingress controller for Kubernetes using NGINX a...
```

查阅[ingress-nginx官方文档](https://github.com/kubernetes/ingress-nginx), 确认与Kubernetes 1.28匹配的Ingress版本: v1.9.0 - v1.12.0, 我选择安装v1.11.4
下载1.11.4版本的ingress-nginx软件包, 这里没有使用helm install直接安装, 因为我们需要修改一些配置
```
helm pull my-ingress/ingress-nginx --version 4.11.4
```
注: 如果下载失败, 可以尝试配系统代理, 让VMware虚拟机走主机代理, 参考[https://blog.xzr.moe/archives/124/](https://blog.xzr.moe/archives/124/)

解压tgz包, 修改配置文件values.yaml
```
tar xf ingress-nginx-4.11.4.tgz

# 修改配置文件values.yaml
sed -i '/registry:/s#registry.k8s.io#swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io#g' ingress-nginx/values.yaml
sed -ri '/digest:/s@^@#@' ingress-nginx/values.yaml
sed -i '/hostNetwork:/s#false#true#' ingress-nginx/values.yaml
sed -i  '/dnsPolicy/s#ClusterFirst#ClusterFirstWithHostNet#' ingress-nginx/values.yaml
sed -i '/kind/s#Deployment#DaemonSet#' ingress-nginx/values.yaml 
sed -i 's/type: LoadBalancer/type: NodePort/' ingress-nginx/values.yaml

说明:
* 镜像改成国内的, 国内由于网络问题无法下载海外镜像
* 注释掉digest, 因为国内镜像可能被重新构建过, digest并不相同
* hostNetwork设为true, 直接监听宿主机80和443端口 （不设置也可以, 通过NodePort访问)
* 如果设置了hostNetwork为true, dnsPolicy建议也改成ClusterFirstWithHostNet, 可以使用k8s内部的service名称解析
* Deployment改成DaemonSet类型, 每个节点部署一个Pod (这一步不是必须的, 你可以用Deployment+nodeSelector调度到指定节点)
* 本地测试, 没有云服务商提供负载均衡服务，LoadBalancer类型改成NodePort
```

为ingress-nginx创建namespace
```
kubectl create namespace ingress-nginx
```

使用helm安装ingress-nginx
```
helm install my-ingress-nginx ./ingress-nginx --namespace ingress-nginx
```

测试
```
# kubectl -n ingress-nginx get pods  -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP               NODE         NOMINATED NODE   READINESS GATES
my-ingress-nginx-controller-2pj8h   1/1     Running   0          51s   192.168.52.102   k8s-slave2   <none>           <none>
my-ingress-nginx-controller-bjp7z   1/1     Running   0          51s   192.168.52.103   k8s-slave3   <none>           <none>
my-ingress-nginx-controller-lk9xj   1/1     Running   0          51s   192.168.52.101   k8s-slave1   <none>           <none>

# kubectl -n ingress-nginx get svc
NAME                                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
my-ingress-nginx-controller             NodePort    10.110.172.220   <none>        80:31481/TCP,443:32374/TCP   63s
my-ingress-nginx-controller-admission   ClusterIP   10.98.166.222    <none>        443/TCP                      63s

从集群内部访问, OK
# curl 10.110.172.220:80
404 Not Found (404是OK的, 说明80端口已LISTEN, Nginx响应了请求) 

访问从节点80端口 OK, 访问主节点80端口显示connection refused
# curl k8s-master1:80
curl: (7) Failed to connect to k8s-master1 port 80: Connection refused
# curl k8s-slave1:80
404 Not Found

通过NodePort访问任意节点, 都是OK的
curl k8s-master1:31481
404 Not Found
curl k8s-slave1:31481
404 Not Found
```

## 卸载应用
首先查看需要卸载的应用
```
helm -n ingress-nginx list
NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
my-ingress-nginx        ingress-nginx   1               2025-03-22 15:45:56.455495532 +0800 CST deployed        ingress-nginx-4.11.4    1.11.4
```
再使用helm uninstall卸载
```
helm -n ingress-nginx uninstall my-ingress-nginx
```

# 搭建私有Helm Repo

准备一台Linux机器(我用的是Rocky Linux 9.5)，用于快速搭建Helm Repo
使用简单的HTTP服务器托管Chart, 实现基本的身份验证, 以下使用Nginx+htpasswd实现

## 安装必要软件
```
yum install nginx httpd-tools -y
```

## 使用htpasswd常见一个包含用户名和密码的文件, 添加认证用户helmuser (password:123456)
```
htpasswd -c /etc/nginx/charts.htpasswd helmuser
```

## 配置Nginx

本地测试环境, 为myhelmrepo.com生成一个自签名证书
```
sudo mkdir -p /etc/nginx/ssl/myhelmrepo.com
cd /etc/nginx/ssl/myhelmrepo.com
```
生成私钥和证书签名请求
```
sudo openssl req -new -newkey rsa:2048 -nodes -keyout myhelmrepo.com.key -out myhelmrepo.com.csr
```
填写CSR表单时, Common Name(CN)应设置为 myhelmrepo.com

使用该CSR生成自签名证书
```
sudo openssl x509 -req -days 365 -in myhelmrepo.com.csr -signkey myhelmrepo.com.key -out myhelmrepo.com.crt
```

创建Nginx配置文件
```
mkdir -p /etc/nginx/sites-available
vim /etc/nginx/sites-available/helmrepo
```
加入如下内容, 使用HTTPS, Basic认证
```
server {
    listen 443 ssl;
    server_name myhelmrepo.com; # 替换为您的服务器域名或IP地址

    ssl_certificate /etc/nginx/ssl/myhelmrepo.com/myhelmrepo.com.crt;
    ssl_certificate_key /etc/nginx/ssl/myhelmrepo.com/myhelmrepo.com.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    location / {
        auth_basic "Restricted Area";
        auth_basic_user_file /etc/nginx/charts.htpasswd;

        root /var/www/html/charts/;
        autoindex on;
    }
}
# HTTP请求重定向到 HTTPS
server {
    listen 80;
    server_name myhelmrepo.com;
    return 301 https://$host$request_uri;
}
```
Nginx主配置文件
```
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 4096;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

启用配置
```
mkdir -p /etc/nginx/sites-enabled/
ln -s /etc/nginx/sites-available/helmrepo /etc/nginx/sites-enabled/helmrepo
```

检查Nginx配置并重启
```
nginx -t
systemctl restart nginx
ss -antp | grep 80
LISTEN    0      511           0.0.0.0:80           0.0.0.0:*     users:(("nginx",pid=2596,fd=7),("nginx",pid=2595,fd=7))
```
测试
```
curl https://myhelmrepo.com -k
401 Authorization Required
curl --user helmuser:123456 https://myhelmrepo.com -k
404 Not Found 
```

上传chart
在开发机器上打包Chart
```
helm package simple-flask-app/
```
把tgz传到/var/www/html/charts/目录下

更新helm repo的索引文件
```
helm repo index /var/www/html/charts/ --merge /var/www/html/charts/index.yaml
```
如果是第一次添加chart, 直接运行
```
helm repo index /var/www/html/charts
```

测试:

# 自定义Helm Chart









# 发布流程
* 部署应用 helm install
* 升级应用 helm upgrade
* 回滚机制 helm rollback





# 参考
【1】 [https://docs.daocloud.io/kpanda/user-guide/helm/](https://docs.daocloud.io/kpanda/user-guide/helm/)
【2】 [使用Helm管理kubernetes应用](https://jimmysong.io/kubernetes-handbook/practice/helm.html)
【3】 [Ingress-Nginx使用指南](https://www.cnblogs.com/yinzhengjie/p/17975829)