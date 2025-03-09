---
layout: next
title: 通过定制initramfs实现从单系统分区到双系统的无缝升级
date: 2025-03-01 16:24:19
categories: Project
tags: Project
---

# 背景
我们的客户有一些本地部署的网关设备(CentOS 7)需要做固件升级。 原有的单系统分区架构缺乏有效的回滚机制, 升级遇到故障后无法回滚, 导致服务中断, 用户体验差

# 目标
设计一种解决方案, 自动将客户的网关设备从单系统分区平滑升级到双系统，同时保证用户配置(IP, DNS, 登录口令)不丢失
整个升级过程对客户完全透明, 无需客户进行额外操作(比如添加磁盘, 创建新机器)

# 方案
| 方案 | 描述 | 优点 | 缺点 |
| --- | ---- | --- | ---- |
| 定制initramfs	| 通过定制initramfs进入紧急模式，预先加载升级包和配置文件到内存，再重建磁盘分区 | 用户无感知, 且备份了旧的配置 | 内存空间有限, 升级包+解压后系统文件需小于内存 |
| 安装ISO | 下载并安装目标系统ISO | 不需要单独出一个升级包 | 客户的配置很难同步, 升级后需要用户做一些手动配置 | 

最终采用定制initramfs方案。 客户虚拟机内存最低是8G, 需保证升级包+解压后系统文件远小于8G即可

# 实现

**流程**
![](image1.png)


**升级包**
升级包就是一个tar包(migration.tar.gz)，客户的VA通过网络下载、解压升级包, 执行其中的脚本自动完成升级。 升级包的内容如下：
```bash
# tar -tvf migration.tar.gz
va.sgdisk                   # 使用sgdisk导出的新系统的分区表文件，支持双系统启动
va.tar.xz                   # 一个压缩包, 把RockyLinux9.4虚拟机导出到OVA，再把OVA文件系统中根分区下的所有文件备份成压缩包
initramfs-convert.img       # 基于Rocky9官方ISO的initramfs定制, 用于启动升级小系统
vmlinuz-convert             # 一个压缩内核, 直接从Rocky9官方ISO的vmlinuz拷贝，用于启动升级小系统
upgrade_to_tiny_system.sh   # 一个Shell脚本，在客户VA上解压升级包后，自动执行这个脚本，切换到升级小系统启动 
```

要点:
* 分区方案采用BIOS/GPT, BIOS是为了兼容老客户, GPT可以支持2T以上磁盘, 可扩展性和性能更好
* 为了使升级包尽可能小, 使用CentOS 7 minimal ISO(900M), 仅安装了必要的组件(Microk8s), 使用xz压缩算法备份
* tar备份目标系统时, 需要`--numeric-owner`保留文件的UID/GID, 以及文件扩展属性xattr(有snap的应用运行状态）

# 难点
备选方案:
通过安装ISO方式  需要手动备份旧的配置

数据完整性何一致性保证

自动化流程设计, 过程对用户透明且无感

# 调试方法
手动进入GRUB rescue模式加载内核, 手动挂载磁盘根分区，查看失败日志
```
set root=(hd0,gpt2)
linux (hd0,gpt2)/vmlinuz-convert root=/dev/mapper/VA-root ro rd.lvm.lv=VA/root
initrd (hd0,gpt2)/initramfs-convert.img
```
先对虚拟机做快照, 如果调试失败通过快照迅速恢复系统

# 问题
Q: 切换到新系统后, 输入正确的用户名和密码也无法登录, console一直打印localhost login
A: 通过手动进入GRUB rescue模式调试, 挂载磁盘日志发现是SELinux的问题, 将目标系统的SELinux设置为disabled后，问题得到解决