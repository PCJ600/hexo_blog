---
layout: next
title: VMware ESXi导出OVA文件中含有ISO文件，如何去除这个ISO
date: 2024-12-17 23:31:42
tags:
---

## 问题描述
我在VMware ESXi上，用官方Rockylinux minimal ISO装了一台Linux机器。导出OVA文件发现大小有3个G。查看OVA文件，发现有个ISO文件占了2G，这个ISO文件用不到，如何删除？
```
# tar tvf va.ova
-rw-r--r-- someone/someone 9876 2024-12-17 02:27 va.ovf
-rw-r--r-- someone/someone  345 2024-12-17 02:27 va.mf
-rw-r--r-- someone/someone 1829634048 2024-12-17 02:27 va-file1.iso
-rw-r--r-- someone/someone 1332535296 2024-12-17 02:28 va-disk1.vmdk
-rw-r--r-- someone/someone       8684 2024-12-17 02:28 va-file2.nvram
```
## 解决方法
Poweroff虚拟机，修改虚拟机配置，把光驱删掉，再重新导出OVA即可 
<!-- more -->
![image1.png](image1.png)

新的OVA大小只有1.3G，且ISO成功删除
```
tar tvf ./va.ova
-rw-r--r-- someone/someone 9226 2024-12-17 02:41 va.ovf
-rw-r--r-- someone/someone  258 2024-12-17 02:41 va.mf
-rw-r--r-- someone/someone 1332317696 2024-12-17 02:42 va-disk1.vmdk
-rw-r--r-- someone/someone       8684 2024-12-17 02:42 va-file1.nvram
```

## VMware ESXi导出OVA的方法
除了在UI上导出，也可以通过Vmware PowerCLI和ovftool导出OVA
```
pwsh
PS> Set-PowerCLIConfiguration -InvalidCertificateAction "Ignore" -Confirm:$false | Out-Null
PS> connect-viserver {your_viserver_ip} -User root -Password {your_password}  
PS> ovftool -dm=thin --noSSLVerify --powerOffSource vi://root:{your_password}@{viserver_ip}//{your_vm_name} va.ova
```

## 参考
[https://duiz.net/3491.html](https://duiz.net/3491.html)