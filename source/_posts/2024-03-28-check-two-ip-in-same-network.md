---
layout: next
title: 判断两个IP地址是否在同一网段(Shell实现)
date: 2024-03-28 19:50:11
categories: Shell
tags: Shell
---

## 代码实现
```bash
#!/bin/bash

# 函数：提取 CIDR 的网络地址和子网掩码
function extract_network() {
    echo $1 | awk -F '/' '{print $1}'
}

function extract_subnet() {
    echo $1 | awk -F '/' '{print $2}'
}

# 函数：将 IP 地址转换为二进制格式
function ip_to_binary() {
    local ip=$1
    local binary=""
    local IFS='.'
    local octets=($ip)
    for octet in "${octets[@]}"; do
        local bin_octet=""
        local num=$octet
        for (( i=0; i<8; i++ )); do
            bin_octet=$((num % 2))$bin_octet
            num=$((num / 2))
        done
        binary+=$bin_octet
    done
    echo $binary
}
```
<!-- more -->

```bash
# 函数：比较两个 IP 地址是否在同一网段
function same_network() {
    network1=$(extract_network $1)
    subnet1=$(extract_subnet $1)
    network2=$(extract_network $2)
    subnet2=$(extract_subnet $2)

    binary1=$(ip_to_binary $network1)
    binary2=$(ip_to_binary $network2)

    # 截取相同长度的二进制子串
    binary1=$(echo $binary1 | cut -c1-$subnet1)
    binary2=$(echo $binary2 | cut -c1-$subnet2)

    if [ "$binary1" == "$binary2" ]; then
        echo "两个 CIDR 在同一网段"
    else
        echo "两个 CIDR 不在同一网段"
    fi
}
```

## 测试
CIDR1="10.206.216.21/24"
CIDR2="10.206.217.10/24"
same_network $CIDR1 $CIDR2
