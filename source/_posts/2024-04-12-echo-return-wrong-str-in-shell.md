---
layout: next
title: Shell函数中使用echo返回了错误字符串问题定位
date: 2024-04-12 20:24:33
categories: troubleshooting
tags:
- troubleshooting
- shell
---

## 问题描述
在Vmware虚拟机上，执行如下Shell代码获取VM类型
```bash
#!/bin/bash

function get_VM_infra() {
    local infra=$(dmidecode -t system | grep "Manufacturer")
    echo "$infra" | grep "VMware" # 判断VM平台类型是否为VMware
    if [ $? -eq 0 ]; then
        echo "VMware"
    else
        echo "Other Type"
    fi
}

infra=$(get_VM_infra)
echo "infra: $infra"
```
预期返回`"VMware"`，实际返回`"Manufacturer: VMware, Inc.\nVMware"`，这个返回的结果是错误的
```
# infra:  Manufacturer: VMware, Inc.
# VMware
```
## 原因分析
<!-- more -->
问题出在`echo "$infra" | grep "VMware"`，**这行代码判断虚拟机类型的时候，也使用了echo，导致返回的字符串多了一行**(Manufacturer: VMware, Inc.)。

## 解决方法
方法1：
`echo "$infra" | grep "VMware"` 改成 `echo "$infra" | grep "VMware" > /dev/null 2>&1`

方法2:
直接定义全局变量保存字符串，回避echo语句的副作用

## 参考
[How to Return String From Bash Function](https://linuxsimply.com/bash-scripting-tutorial/functions/return-values/return-string-function/)
