---
layout: next
title: GRUB菜单不显示问题定位
date: 2024-03-21 19:52:18
categories: troubleshooting
tags:
- Linux
- GRUB
- troubleshooting
---

# 问题描述
Rocky Linux 9.2环境，手工配置了`GRUB_TIMEOUT`为30秒，但重启后发现没有显示菜单，未出现30秒倒计时。

# 调试方法
梳理GRUB启动流程，下载GRUB源码并重新编译、安装、调试，具体步骤如下：
<!-- more -->

# 0. 梳理GRUB启动流程
Linux启动流程参考： [谈谈Linux系统启动流程](https://www.cnblogs.com/quan0311/p/15292110.html)
GRUB启动流程参考下图：(图片非原创，转载)
![](image1.png)

# 1. 确认GRUB版本号
```bash
# rpm -qa | grep grub
grub2-pc-2.06
# grub2-install --version
grub2-install (GRUB) 2.06
```
得到GRUB版本号为2.06
# 2. 下载GRUB源码
```bash
wget https://ftp.gnu.org/gnu/grub/grub-2.06.tar.gz
tar -xzvf grub-2.06.tar.gz
cd grub-2.06
```
# 3. 修改GRUB源码，添加调试日志
修改`grub-core/normal/menu.c`中的函数`run_menu`
* 使用`grub_printf`函数添加日志打印`timeout`，`timeout_style`, `default_entry`
* 使用`grub_millisleep`函数休指定休眠时间(单位毫秒，休眠目的是让日志停留一段时间，方便定位）
![](image2.png)
kernel.img执行流程：`startup.S -> grub_main -> grub_load_normal_mode -> grub_command_execute("normal", 0 ,0) -> grub_show_menu -> show_menu -> run_menu -> ...`
# 4. 编译和安装GRUB
```bash
yum install -y bison gcc flex  # 安装必要依赖
./configure --prefix=/usr	# 笔者的RockyLinux9.2默认应安装到/usr, 其他环境只需后台确认下grub安装的路径，指定对应的prefix即可
make # 编译
make install # 安装
grub2-install /dev/sda # 重新安装GRUB到MBR, 根据你的环境把/dev/sda改成具体的虚拟硬盘设备
```
# 5. 重启后，通过串口查看GRUB日志，确认修改生效
串口日志打印如下：
```
GRUB loading.
Welcome to GRUB!
GET DEFAULT ENTRY: 0
GET TIMEOUT: 1
GET TIMEOUT STYLE: 2
```
发现`timeout`值为1，`timeout_style`为2(TIMEOUT_STYLE_HIDDEN)，不符合预期的30秒设定，查看`/boot/grub2/grub.cfg`，找到了对应的代码段：
![](image3.png)
通过`grub2-editenv`命令查看`menu_auto_hide`这个环境变量的确存在，且值为1，所以匹配了else语句，`timeout`的值被设置了1秒
```
# grub2-editenv - list | grep menu_auto_hide
menu_auto_hide=1
```
查看[Rocky Linux官方文档](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/9/html/considerations_in_adopting_rhel_9/ref_notable-changes-to-boot-loader_assembly_kernel)，找到了`menu_auto_hide`环境变量被设置的原因，以及解决方法：
![](image4.png)

# 参考资料
【1】 [谈谈Linux系统启动流程](https://www.cnblogs.com/quan0311/p/15292110.html)
【2】 [Red hat Customer Portal](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/9/html/considerations_in_adopting_rhel_9/ref_notable-changes-to-boot-loader_assembly_kernel)
