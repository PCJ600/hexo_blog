---
layout: next
title: CLISH logout后没有执行自定义脚本问题的解决方法
date: 2024-08-19 22:34:18
categories: troubleshooting
tags:
- troubleshooting
- CLISH
---

## 问题描述
我有一个On-Premises Appliance使用了[CLISH](https://clish.sourceforge.net/)(思科命令行)，需要对logout操作记录auditlog。我在配置文件root-view.xml里添加了自定义脚本记录auditlog，但是脚本没有执行，auditlog没有记录。

## 解决思路
Google没查到类似案例，毕竟这年头互联网公司谁还用这种命令行呢，只好把CLISH源代码下一份，捋下流程看看解决方法,  先给出我的root-view.xml配置：
<!-- more -->
```xml
	<COMMAND name="logout" help="Logout of the current CLI session">
		<ACTION builtin="clish_close">	<!-- 内置的clish_close方法，用于退出clish -->
		/usr/share/clish/scripts/basecmd.sh release #自定义的脚本，用于记录auditlog
        </ACTION>
	</COMMAND>
```

查看CLISH源码，看下root-view.xml中的command怎么解析的：
<!-- more -->
![](image1.png)


观察if语句发现问题所在：如果builtin存在，就不执行script。 我在配置文件里指定了builtin函数clish_close, 所以自定义的脚本没有执行。

## 解决方法
我想到两种方法：
**方法1：**自行修改CLISH源码, 改C函数clish_close, 添加自定义代码。 这种方法需要自己维护源代码，不太推荐。

**方法2:** 在root-view.xml里自定义action, 通过kill $PPID(直接杀死父进程)的方式退出CLISH。

这种方法不需修改源代码，问题是每次退出会打印一行"Terminated"，无伤大雅。 将root-view.xml配置修改如下：
```xml
	<COMMAND name="logout" help="Logout of the current CLI session">
		<ACTION>	<!-- 去掉内置的clish_close方法 -->
		/usr/share/clish/scripts/basecmd.sh release
		kill $PPID # 杀掉父进程，也就是CLISH主进程
        </ACTION>
	</COMMAND>
```

## CLISH进程模型分析
* 一个主进程clish, 创建一个副线程循环读取用户输入的指令，主进程等待副线程退出;
* 副线程每读取一个指令就创建一个子进程，子进程执行xml里定义的builtin和action，执行完子进程退出。

分析过程如下，通过GDB和ps命令，没啥难度：
```bash
ps aux
 339658 ?        Ss     0:00 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
 339765 pts/0    Ss     0:00  |       \_ -bash
 383959 pts/0    Sl+    0:00  |           \_ clish (有两个线程, 副线程(383960)负责读取输入指令，主线程等待副线程(383960)结束
 384002 pts/0    S+     0:00  |               \_ sh -c  ??        /usr/share/clish/scripts/basecmd.sh release ??        echo "pid: $$, ppid: $PPID" ?? (执行用户自定义action的进程)


ps -T 383959 # (CLISH进程有两个线程)
    PID    SPID TTY      STAT   TIME COMMAND
 383959  383959 pts/0    Sl+    0:00 clish (创建副线程383960，等待其退出)
 383959  383960 pts/0    Sl+    0:00 clish (副线程循环读取用户输入指令, 创建子进程执行自定义builtin和action)

ps ajx | grep clish # (查看相关CLISH进程的PID, 父进程ID, 线程ID)
   PPID     PID    PGID     SID TTY        TPGID STAT   UID   TIME COMMAND
 339765  383959  383959  339765 pts/0     383959 Sl+      0   0:00 clish
 383959  384002  383959  339765 pts/0     383959 S+       0   0:00 sh -c  ??   /usr/share/clish/scripts/basecmd.sh release ??        echo "pid: $$, ppid: $PPID" ??        sleep 1000 ??        kill $PPID ???
```

