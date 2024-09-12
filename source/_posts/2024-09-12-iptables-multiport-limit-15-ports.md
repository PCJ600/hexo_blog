---
layout: next
title: iptables单个multiport规则最多只能指定15个端口问题的解决方法
date: 2024-09-12 21:14:01
categories: iptables
tags: iptables
---

## 问题描述
我想通过iptables允许以下这20个端口通过：
```
iptables -A INPUT -p tcp -i eth0 -m multiport --dports 22,80,443,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46 -j ACCEPT
```
执行后报错，`iptables: too many ports specified`

## 原因
查看[iptables官方文档](https://linux.die.net/man/8/iptables), 发现iptables单条multiports规则最多只支持15个端口。 原文如下： 

<!-- more -->
```
multiport

This module matches a set of source or destination ports. Up to 15 ports can be specified. A port range (port:port) counts as two ports. It can only be used in conjunction with -p tcp or -p udp.
```
## 解决方法
每15个端口新增一条multiport的rule即可, 代码参考:
```bash
#!/bin/bash

all_ports="22,80,443,30,31,32,33,34,35,36,37,38,39,40,41,42,43,44,45,46"
ports_array=($(echo $all_ports | tr ',' ' '))
ports=""
for (( i=0; i<${#ports_array[@]}; i++ )); do
    ports+="${ports_array[$i]},"
    # One iptables multiports rule supports at most 15 ports
    if (( (i + 1) % 15 == 0 || i == ${#ports_array[@]} - 1 )); then
        ports=${ports%,}
        echo "ports: ${ports}"
        sudo iptables -I INPUT -p tcp -m multiport --dports ${ports} -j ACCEPT
        sudo iptables -I INPUT -p tcp -m multiport --sports ${ports} -j ACCEPT
        ports=""
    fi
done
```

## 参考
[https://linux.die.net/man/8/iptables](https://linux.die.net/man/8/iptableshttps://linux.die.net/man/8/iptables)


