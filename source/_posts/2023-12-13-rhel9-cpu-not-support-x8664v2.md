---
layout: next
title: RHEL9 Fatal glibc error, CPU does not support x86-64-v2 解决方法
date: 2023-12-13 19:37:45
categories: Linux
tags: Linux
---

## 问题描述
RHEL 9要求x86_64的CPU支持x86-64-v2，x86-64-v2需要处理器支持 CMPXCHG16B、LAHF-SAHF、POPCNT、SSE3、SSE4.1、SSE4.2、SSSE3 等现代指令集

## 检查CPU是否支持x86-64-v2的方法
```bash
#!/bin/sh
flags=$(cat /proc/cpuinfo | grep flags | head -n 1 | cut -d: -f2)
supports_v2='awk "/cx16/&&/lahf/&&/popcnt/&&/sse4_1/&&/sse4_2/&&/ssse3/ {found=1} END {exit !found}"'
echo "$flags" | eval $supports_v2
if [ $? -eq 0 ]; then
	echo "CPU supports x86-64-v2"
else
	echo "CPU doesn't support x86-64-v2"
fi
```
<!-- more -->
## 参考
【1】[https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/9.0_release_notes/architectures](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/9.0_release_notes/architectures)
【2】[https://unix.stackexchange.com/questions/631217/how-do-i-check-if-my-cpu-supports-x86-64-v2](https://unix.stackexchange.com/questions/631217/how-do-i-check-if-my-cpu-supports-x86-64-v2)
