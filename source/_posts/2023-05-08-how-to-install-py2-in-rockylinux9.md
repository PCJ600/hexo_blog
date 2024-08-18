---
layout: next
title: RockyLinux9 docker中安装python2方法
date: 2023-05-08 21:32:15
categories: Python
tags: Python
---

新增Dockerfile, 内容如下：
<!-- more -->
```
FROM rockylinux:9.1

RUN yum -y install gcc zlib openssl openssl-devel zlib-devel wget

RUN mkdir /py2 && mkdir /usr/local/python2.7

RUN wget -O /py2/Python-2.7.14.tgz https://www.python.org/ftp/python/2.7.14/Python-2.7.14.tgz \
    && tar -xzf /py2/Python-2.7.14.tgz -C /py2/

RUN cd /py2/Python-2.7.14 && ./configure --prefix=/usr/local/python2.7/ --enable-shared \
    && sed -i '219,221 s/^.//' /py2/Python-2.7.14/Modules/Setup \
    && make -j4 && make install

RUN echo "/lib" >> /etc/ld.so.conf && ldconfig \
    && ln -s /usr/local/python2.7/lib/libpython2.7.so.1.0 /lib/libpython2.7.so.1.0 \
    && ln -s /usr/local/python2.7/bin/python /usr/bin/python \
    && rm -rf /py2
```
使用docker build命令构建镜像
```bash
docker build -t [image_repo:image_tag] -f Dockerfile .
```

## 参考
[https://www.jianshu.com/p/bcfaa5ff1cfc](https://www.jianshu.com/p/bcfaa5ff1cfc)
