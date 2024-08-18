---
layout: next
title: Linux设置用户密码XX天后失效的方法
date: 2023-02-13 21:07:27
categories: Linux
tags: Linux
---

1. 新增用户`peter`并设置密码`huawei@123`
```
useradd peter
echo peter:huawei@123 | chpasswd
```
2. 使用`chage`命令设置密码过期时间，这里设置7天后密码过期，过期后立即失效
```
chage -m 0 -M 7 -I 0 peter
```
<!-- more -->

`chage`命令用法参考：
```
chage -l
  -I, --inactive INACTIVE       set password inactive after expiration
                                to INACTIVE
  -l, --list                    show account aging information
  -m, --mindays MIN_DAYS        set minimum number of days before password
                                change to MIN_DAYS
  -M, --maxdays MAX_DAYS        set maximum number of days before password
                                change to MAX_DAYS
```
查看信息
```shell
$ chage -l peter
Last password change                                    : Feb 13, 2023
Password expires                                        : Feb 20, 2023
Password inactive                                       : Feb 20, 2023
Account expires                                         : never
Minimum number of days between password change          : 0
Maximum number of days between password change          : 7
Number of days of warning before password expires       : 7

$ cat /etc/shadow | grep peter
peter:$6$XIgc5bJIYhYHSc$Rf0yw7MD2HRPecEwJr9VlshwzBWre6al2lwsfGeB2uVcq1pQaxz2tyvb2r9RtQCSls1sAddjs3BRUCN8IF8fR.:19401:0:7:7:0::
```
