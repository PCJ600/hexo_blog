---
layout: next
title: Linux开机流程
date: 2025-03-02 17:41:42
tags:
---


POST 上电自检
GRUB 控制权交给引导加载器(GRUB2), 加载/boot下的内核和initrd
Kernel 内核加载到内存, 初始化硬件驱动程序
Initrd 临时根文件系统, 加载必要的模块, 挂载真正根文件系统
RootFS 实际根文件系统挂载完成, 控制权交给/sbin/init
Systemd 初始化系统启动服务
Login 用户登录界面启动, 完成系统启动 


## BIOS阶段
BIOS是按下开机键后第一个运行程序, 硬件检测, 从排在第一位的启动设备中读取MBR



