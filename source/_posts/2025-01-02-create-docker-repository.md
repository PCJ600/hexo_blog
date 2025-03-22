---
layout: next
title: 手把手教你搭建Docker私有仓库
date: 2025-01-02 17:10:58
categories: Docker
tags: Docker
---

<!-- toc -->

<!-- more -->

# 测试环境
Rocky Linux 9.5 x86_64

# 搭建步骤

## 安装Docker
```
yum install -y docker
systemctl start docker
systemctl enable docker
docker --version
Docker version 27.4.0, build bde2b89
```

## 拉取Registry容器
Docker官方提供了一个名为registry的容器镜像，可直接用来运行私有仓库. 先拉取registry镜像
```
docker pull registry:2
```

## 配置HTTPS
生成一个包含SANs的自签名证书. 先创建一个OpenSSL配置文件(openssl.cnf), 指定CN为你的域名(mydockerregistry.com)
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
CN = mydockerregistry.com

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = mydockerregistry.com
DNS.2 = localhost
IP.1 = 127.0.0.1
```
生成证书和密钥
```
mkdir -p /mnt/certs
openssl req -x509 -config openssl.cnf -extensions 'req_ext' -nodes -days 365 -newkey rsa:2048 -keyout /mnt/certs/domain.key -out /mnt/certs/domain.crt
```
按照提示填写信息，确保Common Name(CN)设置私有Registry的域名或IP

## 所有客户端信任证书
在所有docker客户端机器上执行如下命令, 信任证书
```
sudo mkdir -p /etc/docker/certs.d/mydockerregistry.com:5000/
sudo cp /mnt/certs/domain.crt /etc/docker/certs.d/mydockerregistry.com:5000/ca.crt
```

## 配置Basic认证
为了保护私有Registry，可以启用基本的用户名密码验证

创建密码文件, 使用htpasswd工具创建一个用户名和密码文件(用户名:dockeruser, 密码:123456)
```
yum install httpd-tools -y
mkdir -p /mnt/auth
htpasswd -Bc /mnt/auth/htpasswd dockeruser
```

## 启动Registry容器
```
systemctl restart docker
docker run -d \
  --name private-registry \
  -p 5000:5000 \
  --restart=always \
  -v /mnt/registry:/var/lib/registry \
  -v /mnt/certs:/certs \
  -v /mnt/auth:/auth \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  registry:2
```
说明：
* `-d`：后台运行容器
* `-p 5000:5000`：将宿主机的 5000 端口映射到容器的 5000 端口
* `-v /mnt/registry:/var/lib/registry`：将本地目录`/mnt/registry`挂载到容器内的`/var/lib/registry`目录，用于存储镜像数据（请确保该目录存在）
* `--restart=always`: 保证容器在主机重启或意外停止后自动启动


## 测试私有Registry
假设我本地有个镜像flask-app:1.0, 需要推送到私有registry, 操作如下：

登录私有Docker registry
```
docker login -u dockeruser -p 123456 mydockerregistry.com:5000
```
给镜像打标签
```
docker tag flask-app:1.0 mydockerregistry.com:5000/flask-app:1.0
```
再推送镜像
```
docker push mydockerregistry.com:5000/flask-app:1.0
```
最后, 在所有客户端机器上测试拉取镜像功能OK
```
# 信任证书
mkdir -p /etc/docker/certs.d/mydockerregistry.com:5000/
scp root@mydockerregistry.com:/mnt/certs/domain.crt /etc/docker/certs.d/mydockerregistry.com:5000/ca.crt

# 登录
docker login -u dockeruser -p 123456 mydockerregistry.com:5000

# 测试拉取镜像
docker pull mydockerregistry.com:5000/flask-app:1.0
1.0: Pulling from flask-app
486dbf987c66: Pull complete
1da0723265ec: Pull complete
4f4cb1a24c66: Pull complete
c876ae22765e: Pull complete
577bd6ae1def: Pull complete
c9ecf2eab7f4: Pull complete
a0bf88afd1f2: Pull complete
Digest: sha256:5e7112644017b0713e4529de43868fc498c1d2dbdefab236e3d64cc11cd036e0
Status: Downloaded newer image for mydockerregistry.com:5000/flask-app:1.0
mydockerregistry.com:5000/flask-app:1.0
```

# 参考
[https://yeasy.gitbook.io/docker_practice/repository/registry](https://yeasy.gitbook.io/docker_practice/repository/registry)