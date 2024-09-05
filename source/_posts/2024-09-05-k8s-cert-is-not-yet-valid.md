---
layout: next
title: '[MicroK8s] Unable to connect to the server: x509: certificate has expired or is not yet valid 问题定位案例'
date: 2024-09-05 20:17:23
categories: troubleshooting
tags: 
- troubleshooting
- k8s
---

## 问题描述
客户有一台基于Hyper-V创建的RHEL9虚拟机, 安装了MicroK8s, 启动正常。
虚拟机重启后, kubectl报错: `Unable to connect to the server: x509: certificate has expired or is not yet valid`

## 定位过程
这个报错说明MicroK8s apiserver证书的有效时间不对。 首先简单了解下Microk8s证书（以下内容 by ChatGPT）
<!-- more -->
* 在 MicroK8s 中，ca.crt 和 server.crt 是位于 /var/snap/microk8s/current/certs/ 目录下的两个重要证书，作用如下：
* ca.crt：这是集群的根证书颁发机构（CA）证书，用于签署其他所有的证书，包括 API 服务器证书、客户端证书、节点证书等。它是建立集群内部所有组件之间信任的基础  。
* server.crt：这个证书是 Kubernetes API 服务器使用的 TLS 证书，它包括了 API 服务器服务的 DNS 名称和 IP 地址。当使用 kubectl 请求 API 服务器时，会使用到这个证书来确保通信的安全性  。
* 当你使用 kubectl 请求 API 服务器时，kubectl 会使用 ca.crt 中的 CA 证书来验证 server.crt 的有效性。如果 server.crt 证书尚未过期，且由受信任的 CA 签发，kubectl 将接受该证书并继续通信 。
* 如果遇到证书过期或即将过期的问题，可以使用 MicroK8s 的 refresh-certs 命令来刷新证书 。这个命令会帮助更新 CA 证书和重新生成 API 服务器证书，以及其他相关的证书 。

回到问题，先通过`openssl`查看apiserver证书有效时间：
```bash
openssl x509 -in /var/snap/microk8s/current/certs/server.crt -text -noout | grep Not
Not Before: Oct 4 00:01:28 2024 GMT
Not After : Oct 4 00:01:28 2025 GMT
```
再查看当前系统时间：
```bash
timedatectl
      Local time: Thu 2024-09-05 06:52:02 UTC
  Universal time: Thu 2024-09-05 06:52:02 UTC
        RTC time: Thu 2024-09-05 06:52:03
       Time zone: America/New_York (EDT, -0400)
     NTP enabled: yes
NTP synchronized: yes
```
发现问题：**当前系统时间是09-05, 而证书有效开始时间是10-04，是一个未来的时间，所以报`certificate is not yet valid`**

我测了几次，发现每次VM重启后，microk8s都会根据当时系统时间重新生成一次apiserver证书。(只重启k8s不重启机器，不会生成新的证书)

客户环境的证书有效开始时间是10-04，又配置了NTP，这说明重新生成证书的那个时刻，系统时间就是10-04，随后NTP服务启动，系统时间又被改对了而已。 通过启动日志确认这一点：
```bash
dmesg -T | grep rtc #(或者查/var/log/message)
[Thu Sep 5 06:46:09 2024] rtc_cmos 00:01: setting system clock to 2024-10-04T00:01:04 UTC
```
从启动日志中，发现VM的RTC时间果然是10-04，比正确时间快了接近一个月。 总结下这个问题发生的过程：
* Hyper-V没有正确同步VM的RTC时间，VM重启的时刻RTC时间为10-04，这个未来的时间被同步给了系统时间。
* MicroK8s重新生成apiserver证书，证书有效时间设置为系统时间10-04
* chronyd服务启动, 系统时间又被NTP server同步成了正确的时间09-05
* 当前系统时间(09-05)早于证书有效时间(10-04)，所以kubectl报错 `certificate is not yet valid`

至于Hyper-V为什么没有正确同步虚拟机的RTC时间，这个需要Microsoft的支持。 下面给出一个workaround规避这个问题：
## 解决方法
每次启动判断下MicroK8s证书时间是否有效，如果是一个未来的时间，就重新刷下MicroK8s证书。
```bash
CERT_FILE="/var/snap/microk8s/current/certs/server.crt"
not_before=$(date -d "$(openssl x509 -noout -dates -in "$CERT_FILE" | grep "notBefore" | cut -d= -f2)" +%s)
current_time=$(($(date +%s) + 300))

if [ ! -z "$not_before" ] && [ "$current_time" -lt "$not_before" ]; then
	microk8s refresh-certs --cert server.crt
fi
```
注：RTC时间可以不用手动设置，因为配置了NTP server之后一段时间会自动同步

可以添加一个systemd service, 使用`systemctl enable`命令，每次启动的时候执行。service配置文件如下：
```
[Unit]
Description=Refresh k8s certs if cert is not yet valid
After=chronyd.service

[Service]
Type=oneshot
ExecStart=/path/to/your_script.sh

[Install]
WantedBy=multi-user.target
```
