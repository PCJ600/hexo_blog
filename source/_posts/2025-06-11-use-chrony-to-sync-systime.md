---
layout: next
title: Chrony 时间同步
date: 2025-06-11 10:45:27
categories: NTP
tags: NTP
---

## 1. 国内主流 NTP 服务器列表​​

| 服务商 | 国内NTP服务器地址 |
| :-- | :-- |
| 阿里云 | ntp.aliyun.com |
| 腾讯云 |  ntp.tencent.com |
| 华为云 | ntp.huaweicloud.com |
| 清华大学 | ntp.tencent.com |
| 中国国家授时中心 | cn.ntp.org.cn |

## ​​2. 配置 Chrony 使用国内源​​
（1）修改配置文件 `/etc/chrony.conf​​`, ​​替换或添加以下内容​​：
```
# 国内优选源（iburst表示快速初始同步）
server ntp.aliyun.com iburst
server ntp.tuna.tsinghua.edu.cn iburst
server ntp.tencent.com iburst

# 强制使用国内源（忽略配置文件中的其他server）
# pool.ntp.org  # 注释掉默认的国外源

# 调整同步参数
makestep 1 3      # 允许前3次同步时间跳跃（立即修正）
maxdistance 5     # 允许的最大时间偏差（秒）
```
​​（2）重启 Chrony 并验证​​
```
# 重载配置
systemctl restart chronyd

# 等待10秒后检查同步状态
chronyc tracking
chronyc sources -v

# timedatectl 确认时间已同步
timedatectl 
               Local time: Wed 2025-06-11 10:36:14 HKT
           Universal time: Wed 2025-06-11 02:36:14 UTC
                 RTC time: Wed 2025-06-11 02:36:14
                Time zone: Asia/Hong_Kong (HKT, +0800)
System clock synchronized: yes
              NTP service: active
```
<!-- more -->

## 3. 手动立即同步命令​​
```
# 如果时间偏差较大，执行强制同步
chronyd -q "server ntp.aliyun.com iburst" -t 3
# 步进式强制修正（慎用：会突然跳变时间）
chronyc makestep
```
## ​​4. 测试时间源延迟​​
```
# 测试各服务器响应时间
chronyc sourcestats -v

# 手动ping测试
ping -c 4 ntp.aliyun.com
```
选择延迟较低的服务器​​（通常 <50ms 为佳）。
