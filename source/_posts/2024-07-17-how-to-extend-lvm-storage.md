---
layout: next
title: Linux LVM扩容方法实践
date: 2024-07-17 22:23:17
categories: Linux
tags: Linux
---


## 问题描述
VMware Centos环境，根分区为LVM，大小50G，现在需要对根分区扩容。我添加了一块500G的虚拟硬盘(`/dev/sdb`)，如何把这500G扩容到根分区？
<!-- more -->
![](image1.png)
## LVM扩容方法
### 1. 对新磁盘分区
使用`fdisk /dev/sdb`命令，进行交互式分区操作。依次输入n (new一个分区), 输入p (创建primary分区)，输入Partition number(分区编号)为1，其余选项敲回车默认，最后敲w，创建出一个新分区。
![](image2.png)
通过`fdisk -l /dev/sdb` 查看新分区已成功创建
![](image3.png)
### 2. 格式化新分区
先确认根分区的文件格式，先通过`lsblk | grep -w /`查到根分区的逻辑卷为`centos-root`，再通过`blkid`命令得到根分区的文件格式为`xfs`
![](image4.png)
再使用`mkfs -t xfs /dev/sdb1`命令格式化新分区。（如果你的分区是`ext4`格式，就用`mkfs -t ext4 /dev/sdb1`）
![](image5.png)
### 3. 创建物理卷
使用`pvcreate /dev/sdb1`命令创建物理卷
![](image6.png)
### 4. 扩展逻辑卷组
使用`vgs`命令查询逻辑卷组(VG)的名称为`centos`，再使用`vgextend centos /dev/sdb1`命令扩展逻辑卷组
![](image7.png)
`vgextend`执行后，可以看出逻辑卷组大小从511g变成1010.99g，说明扩展成功

### 5. 扩展逻辑卷
使用`lvextend -l +100%FREE /dev/mapper/centos-root`命令，将所有空间扩容到逻辑卷`centos-root`
![](image8.png)
### 6. 调整文件系统大小
对于`xfs`文件系统，使用`xfs_growfs /dev/mapper/centos-root`命令调整文件系统大小
![](image9.png)

最后查看效果，敲 `df -h `，发现根分区大小从50G变成550G，扩容成功！
![](image10.png)

## 参考
[Linux - 通过LVM对磁盘进行动态扩容
](https://www.cnblogs.com/shoufeng/p/10615452.html)
