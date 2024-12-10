---
layout: next
title: VMware虚拟机CPU不支持AVX指令集，但物理CPU支持
date: 2024-12-10 19:55:46
categories:
- VMware
- troubleshooting
tags: 
- VMware
- troubleshooting
---

解决方法：
把虚拟机关机，编辑虚拟机的CPU设置，将CPU虚拟化设置改为硬件模式，再重启虚拟机后，问题解决。

参考：https://blog.csdn.net/hzgnet2021/article/details/134925349