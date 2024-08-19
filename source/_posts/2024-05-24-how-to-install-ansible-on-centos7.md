---
layout: next
title: 如何在centos7上安装ansible
date: 2024-05-24 20:51:10
categories: ansible
tags: ansible
---

先安装python3, 再通过pip安装ansible，最后把ansible安装路径添加到环境变量PATH，如下：
```bash
yum install python3 python3-pip
pip3 install ansible --user
export PATH=$PATH:/usr/local/python3/bin
```
检查安装是否成功
<!-- more -->
```
ansible --version
ansible [core 2.15.12]
  config file = None
  configured module search path = ['/root/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/local/python3/lib/python3.9/site-packages/ansible
  ansible collection location = /root/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/local/python3/bin/ansible
  python version = 3.9.12 (main, Oct 19 2022, 05:17:58) [GCC 4.8.5 20150623 (Red Hat 4.8.5-44)] (/usr/local/bin/python3)
  jinja version = 3.1.2
  libyaml = True
```

## 参考
[https://unix.stackexchange.com/questions/546578/failed-to-install-ansible-on-centos-8](https://unix.stackexchange.com/questions/546578/failed-to-install-ansible-on-centos-8)
