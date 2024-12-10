---
layout: next
title: SSH连接报错, Corrupted MAC on input解决方法
date: 2024-12-10 19:57:32
categories:
- SSH
- troubleshooting
tags:
- SSH
- troubleshooting
---

## 问题描述
客户在windows CMD中SSH连接失败，报错:
```
Corrupted MAC on input
ssh_dispatch_run_fatal: Connection to x.x.x.x port 22: message authentication code incorrect
```
值得注意的是，客户通过别的机器做SSH连接可以成功，使用putty, mobaxterm连接也可以成功

## 原因分析
通过别的机器能做SSH连接，说明SSH服务端本身没有问题，网络连接也正常。 
这个原因一般是SSH客户端和服务器的配置不匹配，使用了不同的MAC(消息认证码)或cipher(加密算法)，导致协商失败。

## 定位思路
分别查看SSH客户端和服务端支持的MAC和cipher列表，先确认配置是否匹配。
<!-- more -->

查看SSH客户端支持的MAC和cipher的方法如下：
```
ssh -Q mac
hmac-sha1
hmac-sha1-96
hmac-sha2-256
hmac-sha2-512
hmac-md5
hmac-md5-96
umac-64@openssh.com
umac-128@openssh.com
hmac-sha1-etm@openssh.com
hmac-sha1-96-etm@openssh.com
hmac-sha2-256-etm@openssh.com
hmac-sha2-512-etm@openssh.com
hmac-md5-etm@openssh.com
hmac-md5-96-etm@openssh.com
umac-64-etm@openssh.com
umac-128-etm@openssh.com

ssh -Q cipher
3des-cbc
aes128-cbc
aes192-cbc
aes256-cbc
aes128-ctr
aes192-ctr
aes256-ctr
aes128-gcm@openssh.com
aes256-gcm@openssh.com
chacha20-poly1305@openssh.com
```

再查看SSH服务端支持的MAC和cipher，有两种方法（无需登录服务端后台)

方法1: 客户端使用nmap测试
```
# nmap --script ssh2-enum-algos -sV -p 22 10.206.216.97

Starting Nmap 6.40 ( http://nmap.org ) at 2024-12-09 22:00 EST
Nmap scan report for 10.206.216.97
Host is up (0.00035s latency).
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.7 (protocol 2.0)
| ssh2-enum-algos:
|   kex_algorithms (10)
|       curve25519-sha256
|       curve25519-sha256@libssh.org
|       ecdh-sha2-nistp256
|       ecdh-sha2-nistp384
|       ecdh-sha2-nistp521
|       diffie-hellman-group-exchange-sha256
|       diffie-hellman-group14-sha256
|       diffie-hellman-group16-sha512
|       diffie-hellman-group18-sha512
|       kex-strict-s-v00@openssh.com
|   server_host_key_algorithms (4)
|       rsa-sha2-512
|       rsa-sha2-256
|       ecdsa-sha2-nistp256
|       ssh-ed25519
|   encryption_algorithms (4)
|       aes256-gcm@openssh.com
|       aes256-ctr
|       aes128-gcm@openssh.com
|       aes128-ctr
|   mac_algorithms (6)
|       hmac-sha2-256-etm@openssh.com
|       umac-128-etm@openssh.com
|       hmac-sha2-512-etm@openssh.com
|       hmac-sha2-256
|       umac-128@openssh.com
|       hmac-sha2-512
|   compression_algorithms (2)
|       none
|_      zlib@openssh.com
MAC Address: 00:0C:29:8F:61:C4 (VMware)
```
重点关注MAC和cipher, 是否和客户端支持的匹配：
* encryption_algorithms（加密算法）： 加密算法用于加密通信双方之间的数据，确保数据的机密性。这些算法可以是对称的（使用相同的密钥加密和解密）或非对称的（使用公钥加密，私钥解密）。SSH中常用的对称加密算法包括aes128-ctr、aes256-ctr等。
* mac_algorithms（消息认证码算法）： 消息认证码（MAC）算法用于确保数据的完整性和真实性，即确保数据在传输过程中没有被篡改，并且确实来自声称的发送者。这些算法通常与加密算法一起使用，为加密的数据提供额外的保护层。常见的MAC算法包括hmac-sha2-256、hmac-sha1等。

其他算法说明：
* kex_algorithms（密钥交换算法）： 密钥交换算法用于在客户端和服务器之间安全地交换加密密钥。这是建立安全会话的第一步，确保后续通信的机密性和完整性。常见的密钥交换算法包括diffie-hellman-group14-sha256、diffie-hellman-group-exchange-sha256等。
* server_host_key_algorithms（服务器主机密钥算法）： 服务器主机密钥算法用于服务器身份验证。服务器使用这些算法生成公钥和私钥对，公钥公开给客户端，私钥用于签名消息以证明服务器的身份。常见的服务器主机密钥算法包括ssh-rsa、ecdsa-sha2-nistp256等。
* compression_algorithms（压缩算法）： 压缩算法用于减少传输数据的大小，从而加快数据传输速度并减少带宽使用。虽然压缩可以提高性能，但它也可能增加CPU的使用率。SSH中常用的压缩算法包括none（不压缩）、zlib等。

方法2: 客户端使用`ssh -vv`, 再根据输出查看MAC和cipher的协商过程
```
ssh -vv root@10.206.216.97

OpenSSH_for_Windows_9.5p1, LibreSSL 3.8.2
debug2: resolve_canonicalize: hostname 10.206.216.97 is address
debug1: Connecting to 10.206.216.97 [10.206.216.97] port 22.
......

debug2: KEX algorithms: curve25519-sha256,curve25519-sha256@libssh.org,ecdh-sha2-nistp256,ecdh-sha2-nistp384,ecdh-sha2-nistp521,diffie-hellman-group-exchange-sha256,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512,diffie-hellman-group14-sha256,ext-info-c,kex-strict-c-v00@openssh.com
debug2: host key algorithms: ssh-ed25519-cert-v01@openssh.com,ecdsa-sha2-nistp256-cert-v01@openssh.com,ecdsa-sha2-nistp384-cert-v01@openssh.com,ecdsa-sha2-nistp521-cert-v01@openssh.com,sk-ssh-ed25519-cert-v01@openssh.com,sk-ecdsa-sha2-nistp256-cert-v01@openssh.com,rsa-sha2-512-cert-v01@openssh.com,rsa-sha2-256-cert-v01@openssh.com,ssh-ed25519,ecdsa-sha2-nistp256,ecdsa-sha2-nistp384,ecdsa-sha2-nistp521,sk-ssh-ed25519@openssh.com,sk-ecdsa-sha2-nistp256@openssh.com,rsa-sha2-512,rsa-sha2-256
debug2: ciphers ctos: chacha20-poly1305@openssh.com,aes128-ctr,aes192-ctr,aes256-ctr,aes128-gcm@openssh.com,aes256-gcm@openssh.com
debug2: ciphers stoc: chacha20-poly1305@openssh.com,aes128-ctr,aes192-ctr,aes256-ctr,aes128-gcm@openssh.com,aes256-gcm@openssh.com
debug2: MACs ctos: umac-64-etm@openssh.com,umac-128-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com,umac-64@openssh.com,umac-128@openssh.com,hmac-sha2-256,hmac-sha2-512
debug2: MACs stoc: umac-64-etm@openssh.com,umac-128-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com,umac-64@openssh.com,umac-128@openssh.com,hmac-sha2-256,hmac-sha2-512
debug2: compression ctos: none,zlib@openssh.com,zlib
debug2: compression stoc: none,zlib@openssh.com,zlib
debug2: languages ctos: 
debug2: languages stoc: 
debug2: first_kex_follows 0 
debug2: reserved 0 
debug2: peer server KEXINIT proposal


debug2: KEX algorithms: curve25519-sha256,curve25519-sha256@libssh.org,ecdh-sha2-nistp256,ecdh-sha2-nistp384,ecdh-sha2-nistp521,diffie-hellman-group-exchange-sha256,diffie-hellman-group14-sha256,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512,kex-strict-s-v00@openssh.com
debug2: host key algorithms: rsa-sha2-512,rsa-sha2-256,ecdsa-sha2-nistp256,ssh-ed25519
debug2: ciphers ctos: aes256-gcm@openssh.com,aes256-ctr,aes128-gcm@openssh.com,aes128-ctr
debug2: ciphers stoc: aes256-gcm@openssh.com,aes256-ctr,aes128-gcm@openssh.com,aes128-ctr
debug2: MACs ctos: hmac-sha2-256-etm@openssh.com,umac-128-etm@openssh.com,hmac-sha2-512-etm@openssh.com,hmac-sha2-256,umac-128@openssh.com,hmac-sha2-512
debug2: MACs stoc: hmac-sha2-256-etm@openssh.com,umac-128-etm@openssh.com,hmac-sha2-512-etm@openssh.com,hmac-sha2-256,umac-128@openssh.com,hmac-sha2-512
debug2: compression ctos: none,zlib@openssh.com
debug2: compression stoc: none,zlib@openssh.com
debug2: languages ctos: 
debug2: languages stoc: 
debug2: first_kex_follows 0 
debug2: reserved 0 

debug1: kex: algorithm: curve25519-sha256
debug1: kex: host key algorithm: ssh-ed25519
debug1: kex: server->client cipher: aes128-ctr MAC: umac-128-etm@openssh.com compression: none
debug1: kex: client->server cipher: aes128-ctr MAC: umac-128-etm@openssh.com compression: none
debug1: expecting SSH2_MSG_KEX_ECDH_REPLY
debug1: SSH2_MSG_KEX_ECDH_REPLY received
debug1: Server host key: ssh-ed25519 SHA256:kQPh28MT+40A8EpxJOcmt+lHMS5QlF1YN/lv05KM+Yg
...
```

## 解决方法
在客户端做ssh连接时, 手动指定服务端支持的MAC或cipher，例如:
```
ssh -m hmac-sha2-512 -c aes128-ctr root@10.206.216.97 # -m指定MAC, -c指定cipher
```
或者修改服务端配置，添加任一个客户端支持的MAC或cipher。在Rocky Linux 9上，可以修改配置文件`/etc/crypto-policies/back-ends/opensshserver.config`

## 参考
【1】[SSH技术白皮书](https://www.h3c.com/cn/d_200805/606213_30003_0.htm)