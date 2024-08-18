---
layout: next
title: 解决Office无法找到应用程序许可证
date: 2020-08-15 18:02:45
categories: Win
tags: Win
---

## 问题描述

打开`office`软件失败，提示无法找到应用程序的许可证。

## 原因

`Software Protection`服务启动失败，可以通过`services.msc`查看该服务的启动状态

## 解决方法

修改注册表，将如下文本复制到文件，文件名改为`software prtection服务.reg`, 双击该文件即可。

<!-- more -->
```
Windows Registry Editor Version 5.00
[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\sppsvc]
"StartProtected"=dword:00000001
"DisplayName"="@%SystemRoot%\\system32\\sppsvc.exe,-101"
"ErrorControl"=dword:00000001
"ImagePath"=hex(2):25,00,53,00,79,00,73,00,74,00,65,00,6d,00,52,00,6f,00,6f,00,\
  74,00,25,00,5c,00,73,00,79,00,73,00,74,00,65,00,6d,00,33,00,32,00,5c,00,73,\
  00,70,00,70,00,73,00,76,00,63,00,2e,00,65,00,78,00,65,00,00,00
"Start"=dword:00000002
"Type"=dword:00000010
"Description"="@%SystemRoot%\\system32\\sppsvc.exe,-100"
"DependOnService"=hex(7):52,00,70,00,63,00,53,00,73,00,00,00,00,00
"ObjectName"="NT AUTHORITY\\NetworkService"
"ServiceSidType"=dword:00000001
"RequiredPrivileges"=hex(7):53,00,65,00,41,00,75,00,64,00,69,00,74,00,50,00,72,\
  00,69,00,76,00,69,00,6c,00,65,00,67,00,65,00,00,00,53,00,65,00,43,00,68,00,\
  61,00,6e,00,67,00,65,00,4e,00,6f,00,74,00,69,00,66,00,79,00,50,00,72,00,69,\
  00,76,00,69,00,6c,00,65,00,67,00,65,00,00,00,53,00,65,00,43,00,72,00,65,00,\
  61,00,74,00,65,00,47,00,6c,00,6f,00,62,00,61,00,6c,00,50,00,72,00,69,00,76,\
  00,69,00,6c,00,65,00,67,00,65,00,00,00,53,00,65,00,49,00,6d,00,70,00,65,00,\
  72,00,73,00,6f,00,6e,00,61,00,74,00,65,00,50,00,72,00,69,00,76,00,69,00,6c,\
  00,65,00,67,00,65,00,00,00,00,00
"DelayedAutoStart"=dword:00000001
"FailureActions"=hex:80,51,01,00,00,00,00,00,00,00,00,00,03,00,00,00,14,00,00,\
  00,01,00,00,00,c0,d4,01,00,01,00,00,00,e0,93,04,00,00,00,00,00,00,00,00,00
[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\sppsvc\Security]
"Security"=hex:01,00,14,80,a0,00,00,00,ac,00,00,00,14,00,00,00,30,00,00,00,02,\
  00,1c,00,01,00,00,00,02,80,14,00,ff,01,0f,00,01,01,00,00,00,00,00,01,00,00,\
  00,00,02,00,70,00,05,00,00,00,00,00,14,00,ff,01,02,00,01,01,00,00,00,00,00,\
  05,12,00,00,00,00,00,18,00,fd,01,0f,00,01,02,00,00,00,00,00,05,20,00,00,00,\
  20,02,00,00,00,00,14,00,9d,01,02,00,01,01,00,00,00,00,00,05,04,00,00,00,00,\
  00,14,00,9d,01,02,00,01,01,00,00,00,00,00,05,06,00,00,00,00,00,14,00,14,00,\
  00,00,01,01,00,00,00,00,00,05,0b,00,00,00,01,01,00,00,00,00,00,05,12,00,00,\
  00,01,01,00,00,00,00,00,05,12,00,00,00
[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\sppsvc\TriggerInfo]
[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\sppsvc\TriggerInfo\0]
"Type"=dword:00000014
"Action"=dword:00000001
"GUID"=hex:da,8a,52,f5,5f,be,14,4f,8a,ef,a9,5d,e7,28,11,61
```

## 参考

[https://blog.csdn.net/jenyzhang/article/details/51867485](https://blog.csdn.net/jenyzhang/article/details/51867485)
