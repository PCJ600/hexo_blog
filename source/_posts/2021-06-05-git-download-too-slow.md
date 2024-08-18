---
layout: next
title: 配置Git代理，解决下载龟速问题
date: 2021-06-05 19:54:18
categories: Git
tags: Git
---

### 背景

git clone龟速或出错，需要配置代理。

### 方法

**首先，你的本地机器必须有socks5代理

进`git bash`，敲如下命令：

```
git config --global http.proxy 'socks5://127.0.0.1:1080'
git config --global https.proxy 'socks5://127.0.0.1:1080'
git config --global http.sslverify false
git config -l
```
<!-- more -->

#### 典型问题

git clone报错: fatal: unable to access XXX.git: OpenSSL SSL_connect: SSL_ERROR_SYSCALL in connection to github.com:443

#### 解决方法
StackOverFlow上能搜到同样问题，参考：[https://stackoverflow.com/questions/49345357/fatal-unable-to-access-https-github-com-xxx-openssl-ssl-connect-ssl-error](https://stackoverflow.com/questions/49345357/fatal-unable-to-access-https-github-com-xxx-openssl-ssl-connect-ssl-error)

进`git bash`，敲如下命令：

```
git config --global --add remote.origin.proxy "127.0.0.1:1087"
```

其中1087为http端口，打开你的ss软件，在设置中查看端口号即可：
![](image1.png)
