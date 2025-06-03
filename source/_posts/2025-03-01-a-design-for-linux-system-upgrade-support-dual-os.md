---
layout: next
title: 通过定制initramfs实现从单系统分区到双系统的无缝迁移
date: 2025-03-01 16:24:19
categories: Linux
tags:
- Linux
- Project
---

# 背景
我们的客户有一些本地部署的网关设备(CentOS 7)需要做固件升级。 原有的单系统分区架构缺乏有效的回滚机制, 升级遇到故障后无法回滚, 导致服务中断, 用户体验差

# 目标
设计一种解决方案, 自动将客户的网关设备从单系统分区平滑升级到双系统，同时保证用户配置(IP, DNS, 登录口令)不丢失
整个升级过程对客户完全透明, 无需客户进行额外操作(比如添加磁盘, 创建新机器)

# 方案
设计如下方案:
| 方案 | 描述 | 优点 | 缺点 |
| --- | ------ | ------ | ------ |
| 定制initramfs	| 通过定制initramfs进入紧急模式，预先加载升级包和配置文件到内存，再重建磁盘分区 | 用户无感知, 且备份了旧的配置 | 内存空间有限, 升级包+解压后系统文件需小于内存 |
| 安装ISO | 下载并安装目标系统ISO | 不需要单独出一个升级包 | 客户的配置很难同步, 升级后需要用户做一些手动配置 | 

<!-- more -->

最终采用定制initramfs方案。 客户虚拟机内存最低配置是8G, 需保证initramfs阶段的升级包+解压后系统文件小于8G

测试结果: c7最小化镜像900M, 安装后磁盘占用1.3G; 安装Microk8s后磁盘占用3G左右, 升级包1.7G, initramfs用掉4G内存, 远小于8G, 可行


# 实现

![](image1.png)

**目标系统分区设计**
```
# lsblk
NAME         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda            8:0    0   10G  0 disk
├─sda1         8:1    0    2M  0 part
├─sda2         8:2    0    1G  0 part /boot
└─sda3         8:3    0  8.8G  0 part
  ├─VA-root  253:0    0  7.8G  0 lvm  /
  ├─VA-image 253:1    0  512M  0 lvm  /image
  └─VA-data  253:2    0  512M  0 lvm  /data
```

**升级包设计**
| 内容 | 描述 |
| --- | --- |
| 目标系统的分区表文件(sgdisk) | 使用sgdisk导出的新系统的分区表文件，支持双系统启动 |
| 目标系统文件的压缩包(.tar.xz) | 把虚拟机导出到OVA，再把VMDK根文件系统中所有文件备份成压缩包 |
| initramfs-convert.img | 基于官方ISO的initramfs定制 |
| vmlinuz-convert | 一个压缩内核, 直接从官方ISO取 |
| upgrade.sh | 二阶段执行, 一阶段在客户机上执行, 下载升级包, 进入紧急模式; 二阶段在initramfs执行, 完成分区重建, 安装和启动目标系统 | 

实现细节:
* 分区方案采用BIOS/GPT, BIOS是为了兼容老客户, GPT可以支持2T以上磁盘, 可扩展性和性能更好
* tar备份目标系统时, 需要`--numeric-owner`保留文件的UID/GID, 以及文件扩展属性xattr(有snap的应用运行状态）

代码参考: [https://github.com/PCJ600/os_migrate](https://github.com/PCJ600/os_migrate)

# 难点
initramfs空间有限, 需要尽可能压缩升级包
* 使用CentOS 7官方的minimal ISO做镜像(900M)
* 只分配必要磁盘空间, 等客户升级成功后动态分配剩余空间, 使升级包尽可能做小
* 导出OVA前, 清理临时文件，日志文件, 禁用交换分区
* 使用压缩比率高的压缩算法(xz)


# 调试方法
1. 手动进入GRUB rescue模式加载内核, 手动挂载磁盘根分区，查看失败日志
```
set root=(hd0,gpt2)
linux (hd0,gpt2)/vmlinuz-convert root=/dev/mapper/VA-root ro rd.lvm.lv=VA/root
initrd (hd0,gpt2)/initramfs-convert.img
```
2. 先对虚拟机做快照, 如果调试失败通过快照迅速恢复系统

# 调试问题
Q: 切换到新系统后, 输入正确的用户名和密码也无法登录, console一直打印localhost login
A: 通过手动进入GRUB rescue模式调试, 挂载磁盘日志发现是SELinux的问题, 将目标系统的SELinux设置为disabled后，问题得到解决
