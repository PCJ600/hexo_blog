---
layout: next
title: Rocky Linux 9安装RabbitMQ
date: 2024-12-18 21:26:31
categories: RabbitMQ
tags: RabbitMQ
---

# 环境准备
Rocky Linux 9

# 安装步骤

##  安装Signing Keys(签名密钥)
```
rpm --import https://github.com/rabbitmq/signing-keys/releases/download/2.0/rabbitmq-release-signing-key.asc
rpm --import https://dl.cloudsmith.io/public/rabbitmq/rabbitmq-erlang/gpg.E495BB49CC4BBE5B.key
rpm --import https://dl.cloudsmith.io/public/rabbitmq/rabbitmq-server/gpg.9F4587F226208342.key
```
<!-- more -->

## 创建repo文件
在/etc/yum.repos.d/目录下创建一个名为rabbitmq.repo的文件，并添加以下内容：
```
# In /etc/yum.repos.d/rabbitmq.repo

##
## Zero dependency Erlang RPM
##

[modern-erlang]
name=modern-erlang-el9
# uses a Cloudsmith mirror @ yum1.novemberain.com.
# Unlike Cloudsmith, it does not have traffic quotas
baseurl=https://yum1.novemberain.com/erlang/el/9/$basearch
repo_gpgcheck=1
enabled=1
gpgkey=https://dl.cloudsmith.io/public/rabbitmq/rabbitmq-erlang/gpg.E495BB49CC4BBE5B.key
gpgcheck=1
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
pkg_gpgcheck=1
autorefresh=1
type=rpm-md

[modern-erlang-noarch]
name=modern-erlang-el9-noarch
# uses a Cloudsmith mirror @ yum1.novemberain.com.
# Unlike Cloudsmith, it does not have traffic quotas
baseurl=https://yum1.novemberain.com/erlang/el/9/noarch
repo_gpgcheck=1
enabled=1
gpgkey=https://dl.cloudsmith.io/public/rabbitmq/rabbitmq-erlang/gpg.E495BB49CC4BBE5B.key
       https://github.com/rabbitmq/signing-keys/releases/download/2.0/rabbitmq-release-signing-key.asc
gpgcheck=1
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
pkg_gpgcheck=1
autorefresh=1
type=rpm-md

[modern-erlang-source]
name=modern-erlang-el9-source
# uses a Cloudsmith mirror @ yum1.novemberain.com.
# Unlike Cloudsmith, it does not have traffic quotas
baseurl=https://yum1.novemberain.com/erlang/el/9/SRPMS
repo_gpgcheck=1
enabled=1
gpgkey=https://dl.cloudsmith.io/public/rabbitmq/rabbitmq-erlang/gpg.E495BB49CC4BBE5B.key
       https://github.com/rabbitmq/signing-keys/releases/download/2.0/rabbitmq-release-signing-key.asc
gpgcheck=1
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
pkg_gpgcheck=1
autorefresh=1


##
## RabbitMQ Server
##

[rabbitmq-el9]
name=rabbitmq-el9
baseurl=https://yum1.novemberain.com/rabbitmq/el/9/$basearch
repo_gpgcheck=1
enabled=1
# Cloudsmith's repository key and RabbitMQ package signing key
gpgkey=https://dl.cloudsmith.io/public/rabbitmq/rabbitmq-server/gpg.9F4587F226208342.key
       https://github.com/rabbitmq/signing-keys/releases/download/2.0/rabbitmq-release-signing-key.asc
gpgcheck=1
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
pkg_gpgcheck=1
autorefresh=1
type=rpm-md

[rabbitmq-el9-noarch]
name=rabbitmq-el9-noarch
baseurl=https://yum1.novemberain.com/rabbitmq/el/9/noarch
repo_gpgcheck=1
enabled=1
# Cloudsmith's repository key and RabbitMQ package signing key
gpgkey=https://dl.cloudsmith.io/public/rabbitmq/rabbitmq-server/gpg.9F4587F226208342.key
       https://github.com/rabbitmq/signing-keys/releases/download/2.0/rabbitmq-release-signing-key.asc
gpgcheck=1
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
pkg_gpgcheck=1
autorefresh=1
type=rpm-md

[rabbitmq-el9-source]
name=rabbitmq-el9-source
baseurl=https://yum1.novemberain.com/rabbitmq/el/9/SRPMS
repo_gpgcheck=1
enabled=1
gpgkey=https://dl.cloudsmith.io/public/rabbitmq/rabbitmq-server/gpg.9F4587F226208342.key
gpgcheck=0
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
pkg_gpgcheck=1
autorefresh=1
type=rpm-md
```

## 安装依赖
```
dnf update -y
sudo dnf install -y socat logrotate
sudo dnf install -y erlang
```
校验Erlang
```
erl
Erlang/OTP 27 [erts-15.2] [source] [64-bit] [smp:2:2] [ds:2:2:10] [async-threads:1] [jit:ns]

Eshell V15.2 (press Ctrl+G to abort, type help(). for help)
1>
```

## 安装RabbitMQ
```
dnf install rabbitmq-server
```

## 启动RabbitMQ, 设置开机自启动
```
systemctl start rabbitmq-server
systemctl enable rabbitmq-server
```

## 配置RabbitMQ

### 启动admin页面插件
```
rabbitmq-plugins enable rabbitmq_management
```
### 添加管理员admin
RabbitMQ自带一个guest用户，默认用户名和密码都是guest。 出于安全考虑，建议创建新的管理员账户并删除或禁用guest账户。  
```
rabbitmqctl delete_user guest
rabbitmqctl add_user admin your_admin_password
rabbitmqctl set_user_tags admin administrator
rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
```
其中, your_admin_password是你为管理员账户设置的密码，/是默认的虚拟主机名称。 操作完成后查看用户列表：
```
sudo rabbitmqctl list_users
Listing users ...
user    tags
admin   [administrator]
```
访问管理页面
```
http://<your_ip_address>:15672/
```

## 示例: 使用python的pika库操作rabbitmq
```
pip3 install pika
```
send.py
```
import pika

credentials = pika.PlainCredentials("admin", "your_password")
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost', credentials=credentials))
channel = connection.channel()
channel.queue_declare(queue='hello')
channel.basic_publish(exchange='',
                      routing_key='hello',
                      body='Hello World!')
print(" [x] Sent 'Hello World!'")
connection.close()
```
receive.py
```
#!/usr/bin/env python
import pika

credentials = pika.PlainCredentials("admin", "your_password@xdr")
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost', credentials=credentials))

channel = connection.channel()

channel.queue_declare(queue='hello')

def callback(ch, method, properties, body):
    print(" [x] Received %r" % body)

channel.basic_consume(queue="hello", on_message_callback=callback, auto_ack=True)

print(' [*] Waiting for messages. To exit press CTRL+C')
channel.start_consuming()
```

## 参考
【1】 [RockyLinux 安装RabbitMQ](https://www.cnblogs.com/eagle6688/p/17437095.html)
【2】 [使用python的pika库来操作rabbitmq](https://pengpengxp.github.io/archive/before-2018-11-10/2016-12-05-pika-and-rabbitmq.html) pus
