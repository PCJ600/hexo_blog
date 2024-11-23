---
layout: next
title: keepalived双机热备方案实现Nginx高可用
date: 2024-11-23 13:56:07
tags:
---

## 问题描述
只用一台Nginx做反向代理，如果这台Nginx出现故障(比如宕机)，则服务不可用。<br/>

以下给出keepalived双机热备方案实现Nginx高可用的方法。先介绍几个概念： <br/>

## 高可用
高可用（High Availability）是指系统或服务能够在面对硬件故障、软件崩溃、网络问题等各种故障情况下，仍然保持正常运行或快速恢复的能力，以减少服务中断时间，确保业务连续性和数据完整性。

## 双机热备
指一台服务器提供服务，另一台作为备用。当一台服务器不可用时另一台就自动顶上去。

## keepalived
一个开源的高可用解决方案，通过VRRP协议实现故障转移，避免单点故障导致的服务中断。

## keepalived双机热备方案实现Nginx高可用的步骤
<!-- more -->

### 准备两台Nginx环境
安装两台Linux虚拟机，每台虚拟机安装Nginx([如何安装Nginx](https://pcj600.github.io//2024/1109163902.html))
* 192.168.52.200 (Nginx1)
* 192.168.52.201 (Nginx2)

修改Nginx.conf, 给Nginx1,Nginx2分别添加一个简单的主页
Nginx1
```
    server {
        listen 80;
        server_name localhost;
        location / {
            default_type text/html;
            return 200 '<h1>welcome to Nginx1</h1>';
        }
		# ...
    }
```
Nginx2
```
    server {
        listen 80;
        server_name localhost;
        location / {
            default_type text/html;
            return 200 '<h1>welcome to Nginx2</h1>';
        }
		# ...
    }
```
启动Nginx
```
systemctl start nginx
```

### 安装并配置Keepalived
Nginx1,Nginx2都安装keepalived, 如下:
```
yum -y install keepalived
```

修改Nginx1的keepalived主配置文件`/etc/keepalived/keepalived.conf`
```
vrrp_script chk_http_port {
    script "/usr/local/bin/keepalived_check_nginx.sh" #心跳检测脚本
    interval 2 #检测脚本执行的间隔，单位是秒）
    weight 2   #权重
}

vrrp_instance VI_1 {
    state BACKUP            # 指定keepalived的角色，MASTER为主，BACKUP为备
    interface ens33         # 指定vrrp通讯的网卡, ifconfig查下你的网卡名
    virtual_router_id 66    # 虚拟路由编号，主从要一致
    priority 100            # 优先级，数值越大，获取处理请求的优先级越高
    advert_int 1            # 检查间隔，默认为1s(vrrp组播周期秒数)
    nopreempt               # 设置为不抢占。这个配置只能在BACKUP主机上面设置
    #授权访问
    authentication {
        auth_type PASS      #设置验证类型和密码，MASTER和BACKUP必须使用相同的密码
        auth_pass peter
    }
    track_script {
        chk_http_port       # 调用检测脚本
    }
    virtual_ipaddress {
        192.168.52.199      # 设置VIP
    }
}
```
Nginx2直接复用Nginx1的keepalived.conf即可<br/>
说明:
* virtual_router_id指虚拟路由编号，主从要一致。
* 如果需要主节点恢复后，VIP再转移到主节点，需要设置为state MASTER；
* 如不需要抢占，可以设置为state BACKUP, 同时设置nopreempt; 对于多个从节点场景，发生故障转移后，根据priority选取MASTER节点。

### 编写检测脚本
编写`keepalived_check_nginx.sh`，判断Nginx是否启动。如果Nginx没有启动并且重启也失败，就停止keepalived服务，进行VIP转移
```
cmd=`ps -C nginx --no-header |wc -l`
if [ $cmd -eq 0 ];then
    systemctl start nginx
    if [ `ps -C nginx --no-header |wc -l` -eq 0 ];then
        killall keepalived
    fi
fi
```
把脚本分别传到Nginx1和Nginx2的`/usr/local/bin`路径

### 配置防火墙
VRRP使用组播地址, 必须配置相应firewall规则或关闭firewall，否则会出现脑裂。
添加规则
```
firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 --in-interface ens33 --destination 224.0.0.18 --protocol vrrp -j ACCEPT
firewall-cmd --direct --permanent --add-rule ipv4 filter OUTPUT 0 --out-interface ens33 --destination 224.0.0.18 --protocol vrrp -j ACCEPT
firewall-cmd --reload
```
或者直接关闭firewall
```
systemctl disable firewalld --now
```

### 启动keepalived，开始测试

#### 测试Nginx1, Nginx2可以正常访问
```
curl 192.168.52.200
Welcome to Nginx1
curl 192.168.52.201
Welcome to Nginx2
```

#### 启动Nginx1和Nginx2的keepalived
```
systemctl start keepalived
```
比如，我先启动Nginx1的keepalived，再启动Nginx2，那么在Nginx1上可以看到VIP(192.168.52.199), Nginx2查不到VIP
```
# ip a
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:36:d4:a9 brd ff:ff:ff:ff:ff:ff
    inet 192.168.52.200/24 brd 192.168.52.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet 192.168.52.199/32 scope global ens33
       valid_lft forever preferred_lft forever
```

#### 通过VIP访问, 可以返回Nginx1的主页
```
curl 192.168.52.199
welcome to Nginx1
```

#### 在Nginx1构造故障, 再通过VIP可以访问Nginx2的主页
```
# 先构造Nginx1故障
curl 192.168.52.199
welcome to Nginx2
```
构造故障方法，可以是先stop nginx，再mv Nginx主程序

#### 再恢复Nginx1(重启Nginx和keepalived)，访问VIP，仍返回Nginx2
```
curl 192.168.52.199
welcome to Nginx2
```
这是符合预期的，因为我们在配置文件里设置了BACKUP和nopreempt

#### 最后构造Nginx2故障，访问VIP，可以返回Nginx1的主页
```
curl 192.168.52.199
welcome to Nginx1
```

## keepalived故障转移原理概述
* 基于VRRP（Virtual Router Redundancy Protocol）协议，通过在多个服务器之间共享一个虚拟IP地址来实现。
* 监控到主服务器（Master）出现故障时，备份服务器（Backup）会检测到并自动接管虚拟IP，继续提供服务。
* 这个过程对客户端是透明的，确保了服务的连续性和高可用性。

## 参考
【1】[nginx+keepalived高可用配置笔记](https://www.cnblogs.com/asker009/p/15010773.html)
【2】[https://docs.nginx.com/nginx/admin-guide/high-availability/ha-keepalived/](https://docs.nginx.com/nginx/admin-guide/high-availability/ha-keepalived/)