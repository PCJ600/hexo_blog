---
layout: next
title: 判断虚拟机类型是VMware,AWS,Azure的方法
date: 2023-11-23 19:35:23
categories: VMware
tags: VMware
---
使用`dmidecode`，如下：
```bash
# AWS：
dmidecode -t system | grep Manufacturer
        Manufacturer: Amazon EC2
# VMware：
dmidecode -t system | grep Manufacturer
        Manufacturer: VMware, Inc
# Azure:
dmidecode -t system | grep Manufacturer
        Manufacturer: Microsoft Corporation
```
