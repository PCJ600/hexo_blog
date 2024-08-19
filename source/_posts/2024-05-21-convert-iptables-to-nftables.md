---
layout: next
title: iptables规则转换为nftables的方法
date: 2024-05-21 20:50:00
categories: iptables
tags: iptables
---

使用预装的`iptables-translate`程序即可，例如：
```bash
# iptables-translate -A INPUT -p icmp --icmp-type time-exceeded -j ACCEPT
nft add rule ip filter INPUT icmp type time-exceeded counter accept
```
<!-- more -->

`nftables`默认没有内置的链，可以自己新增
```bash
nft flush ruleset
nft add table ip filter
nft flush chain ip filter INPUT
nft add chain ip filter INPUT "{type filter hook input priority 0; policy drop; }"
nft add chain ip filter OUTPUT "{type filter hook output priority 0; policy accept; }"
nft add chain ip filter FORWARD "{type filter hook forward priority 0; policy accept; }"
nft list ruleset
```
## 参考
* [https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/9/html/configuring_firewalls_and_packet_filters/getting-started-with-nftables_firewall-packet-filters](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/9/html/configuring_firewalls_and_packet_filters/getting-started-with-nftables_firewall-packet-filters)
* [https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/configuring_firewalls_and_packet_filters/getting-started-with-nftables_firewall-packet-filters#supported-nftables-script-formats_writing-and-executing-nftables-scripts](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/configuring_firewalls_and_packet_filters/getting-started-with-nftables_firewall-packet-filters#supported-nftables-script-formats_writing-and-executing-nftables-scripts)
