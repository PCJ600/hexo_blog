---
layout: next
title: 使用qemu-img把vmdk转成qcow2
date: 2024-06-28 21:58:59
categories: VMware
tags: VMware
---


```bash
qemu-img convert -c -f vmdk -O qcow2 vm.vmdk vm.qcow2
```
