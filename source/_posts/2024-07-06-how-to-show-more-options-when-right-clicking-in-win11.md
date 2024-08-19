---
layout: next
title: Win11右键默认显示更多选项的方法
date: 2024-07-06 22:07:08
categories: Win
tags: Win
---

## 问题描述
win11系统默认右键菜单显示选项太少，每次需要点一下“显示更多选项”才能得到想要内容。比方说我用notepad++打开一个文档，在win11上要先点一下"显示更多选项“，再选择用notepad++打开，操作非常反人类。
<!-- more -->
![](image1.png)
## Win11右键默认显示更多选项的方法
1、点击开始菜单 -> 打开Windows PowerShell (管理员身份运行)
![](image2.png)
2、在Windows PowerShell里输入`reg.exe add "HKCU\Software\Classes\CLSID\{86ca1aa0-34aa-4e8b-a509-50c905bae2a2}\InprocServer32" /f /ve` ，敲Enter执行
![](image3.png)
3、重启win11，发现修改生效
![](image4.png)

