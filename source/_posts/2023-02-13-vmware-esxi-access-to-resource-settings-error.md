---
layout: next
title: VMware Esxi Access to resource settings on the host is restricted to the server...解决方法 
date: 2023-02-13 21:05:42
categories: VMware
tags: VMware
---

## 解决方法
1. 先enable ESXI主机的SSH连接，参考：https://www.uccollabing.com/2016/05/12/enable-ssh-on-vsphere-esxi/
2. 再重启vpxa服务，如下：
```bash
/etc/init.d/vpxa stop ; /etc/init.d/hostd restart

删掉 /etc/vmware/vpxa/vpxa.cfg 文件中 <vpxa/> 的内容

/etc/init.d/vpxa start
```
<!-- more -->
## 参考
[https://www.uccollabing.com/esxi-access-to-resource-settings-on-the-host-is-restricted-to-the-server-that-is-managing-it/](https://www.uccollabing.com/esxi-access-to-resource-settings-on-the-host-is-restricted-to-the-server-that-is-managing-it/)



