---
layout: next
title: Linux下生成core dump文件
date: 2020-02-28 21:07:41
tags: Linux
---

## 问题描述
Linux上运行C程序发生段错误后，没有core文件生成，调试不便。
<!-- more -->

## 生成core文件步骤
1. 敲ulimit -a，查看系统core文件大小限制，如第一行core file size值为0，表示没打开core文件设置
![](image1.png)
2. 敲ulimit -c [kbytes], 设置系统允许生成的core文件大小, 如：
	```
	ulimit -c 1024        设置core文件最大为1024K
	ulimit -c unlimited   不限制core文件大小
	ulimit -c 0           不生成core文件
	```
3. 运行C程序，段错误后，在当前目录生成core文件。
![](image2.png)

**问题:**
多次运行程序发生段错误后，新生成的core文件会把旧的core文件覆盖，怎么区分并保留多个core文件?

**解决方法:**
敲 echo 1 > /proc/sys/kernel/core_uses_pid, 将每次产生的core文件的文件名中是否添加pid作为扩展。如果添加则文件内容为1，反之为0。
![](image3.png)
如上图，两次coredump后，会根据pid生成不同的core文件。

**指定core文件的输出格式和路径**
```
echo /path/to/core.%t.%e.%p > /proc/sys/kernel/core_pattern
```
