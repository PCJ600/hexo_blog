---
layout: next
title: '使用tar备份Linux系统时需要添加--numeric-owner参数'
date: 2025-03-02 14:27:01
categories: Linux
tags: Linux
---

## 前言
在Linux双系统升级过程中, 需要先备份新系统的所有文件得到升级包, 然后在需要升级的机器上解压升级包, 完成升级. tar是Linux系统最常用的备份工具之一。
然而, 在这种跨系统的备份和迁移中, 如果没有正确地处理文件所有者信息, 就会导致权限混乱, 升级后出现一些严重问题, 例如用户无法登录。

我在实际项目中遇到了这个问题。 以下说明为什么使用tar备份Linux系统时需要添加`--numeric-owner`参数

## tar的`--numeric-owner`参数是什么
`--numeric-owner`是tar的一个选项, 用于在打包或解包时, 保留文件的UID和GID, 而不是直接映射当前系统的用户名称和组名称

## 为什么需要`--numeric-owner`参数
因为在跨系统迁移时，源系统和目标系统的用户配置可能是不一致的，例如：
* 目标系统有一个用户alice(UID=1002), 源系统虽然也有用户alice, 但UID=1000
* 这种情况很常见, 比如说目标系统和原系统的OS不一致, 或者在目标系统上新增了一些用户, 造成这种不一致
* 如果备份时没有使用`--numeric-owner`, 源系统解压了tar包后得到的文件UID是1000; 升级到目标系统后, alice(UID=1002)并不是UID=1000的文件所有者, 就会出现权限错误的问题

## 演示案例
以下通过演示, 说明备份系统时忽略`--numeric-owner`时存在的问题

<!-- more -->

假设有两个Linux系统:
* 源系统 A: 有一个alice用户(UID=1000)
* 目标系统 B: 有一个alice用户(UID=1002)

1、在目标系统上创建文件并压缩
```
# echo "hello" > example.txt
# chown alice:alice example.txt
# ls -l example.txt
-rw-r--r--. 1 alice alice 6 Mar  2 15:03 example.txt
# ls -l --numeric example.txt
-rw-r--r--. 1 1002 1002 6 Mar  2 15:03 example.txt 

tar -zcvf example.tar.gz example.txt
```

2、把压缩包传到源系统上

3、在源系统上解压文件(未使用--numeric-owner)
```
# ls -alh example.txt
-rw-r--r--. 1 alice alice 6 Mar  2 02:03 example.txt
# ls -n example.txt
-rw-r--r--. 1 1000 1000 6 Mar  2 02:03 example.txt
```
可以看到, 解压后的文件UID被错误地改成1000, 这样升级到目标系统会有权限问题 

4、在源系统上解压文件(使用--numeric-owner)
```
# tar --numeric-owner -xzvf example.tar.gz
# ls -alh example.txt
-rw-r--r--. 1 1002 1002 6 Mar  2 02:03 example.txt
# ls -n example.txt
-rw-r--r--. 1 1002 1002 6 Mar  2 02:03 example.txt
```
现在, 文件UID=1002被正确保留, 升级到目标系统后不会出现权限问题

## 参考
[https://help.ubuntu.com/community/BackupYourSystem/TAR](https://help.ubuntu.com/community/BackupYourSystem/TAR)