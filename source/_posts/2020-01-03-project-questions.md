---
layout: next
title: 项目问题梳理
date: 2020-01-02 17:56:30
categories: interview
tags: interview
---

<!-- toc -->
<!-- more -->

# 项目: Trend Micro - Service Gateway

## 简单介绍下你的项目
我参与的项目是一个服务网关, 这个服务网关由两部分组成
一个部署在客户网络的虚拟设备, 给趋势本地的安全产品提供微服务, 减少了客户的带宽消耗 
一个云端的APP, 包括一个前后端, 用于管理连接的虚拟设备
我负责虚拟设备的特性开发工作, 主要贡献是设计了一个虚拟设备固件的一键式升级方案, 解决了旧方案中存在的开发维护难, OS迁移困难, 缺乏回滚机制的3个问题
目前有6k+企业客户, 1w+台虚拟设备使用我的升级方案

This project is called Service Gateway. A Service Gateway consists of two parts.
A virtual appliance deployed in corporate network, provide service for Trend On-premises products, reducing customers' bandwidth consumption.
A cloud-based APP, including a frontend and a backend, used to manage connected virtual appliances.
I was responsible for feature development of the virtual appliance. My contribution is to design an online upgrade solution for virtual appliance,
which greatly improved customer experience and development efficiency.
Currently, there are over 6k customers and 10k virtual appliances using my upgrade solution.

## 说下升级方案怎么做的
设计这个方案的目的是, 解决旧的增量升级方案中存在的开发维护难的问题. 因为增量升级需要维护两个版本间差异, 随着时间推移, 历史代码逐渐累积, 我接手项目时, 增量升级的代码有800M, 几乎处于失控状态
为了解决这个问题, 我设计了一个双系统分区的全量升级方案, 分为两个关键步骤:
第一步是做一个全量升级包, 具体方法是: 
* 基于Rocky最小发行版定制虚拟设备的ISO, 需要重新分区, 通过LVM划出两个系统分区.
* 用ISO安装虚拟机, 导出虚拟机的OVA文件。 把所有磁盘文件打包, 得到全量升级包

第二步是实现在线升级。 我开发了一个升级模块, 运行在虚拟设备上。 升级模块包含一个Python的后台进程和一个Cronjob脚本
* Python进程负责和云端通信, 通过订阅AWS IoT消息队列, 接收后端发送的升级消息, 写一个task_file到磁盘, 通知Cronjob脚本处理
* Cronjob脚本读取这个task_file, 执行真正的升级任务, 包括:
	* 下载升级包, 解压升级包到备用分区
	* 把当前的用户配置同步到备用分区
	* 通过grubby添加启动项, 把备用分区设为默认启动项
	* 整机重启, 恢复配置

I designed this solution to solve problems in old incremental upgrade solution. Since incremental upgrade requires maintaining differences between two versions,
the accumulation of historical code made the system complex and difficult to manage. 
When I took over this project, the codebase for incremental upgrade had grown to 800 MB, it's becoming unmanagable.
So, I designed a full-upgrade solution with dual system, which consists of two steps:
Firstly, we need to build a full upgrade package.
Customize an ISO image for virtual appliance based on minimal Rocky Linux. We need to re-partition the disk and use LVM to create two system partitions.
Use ISO to install a VM, then export VM to OVA. Pack all disk files of OVA and we'll get the full upgrade package
Secondly, we need to upgrade online. I developed an upgrade module. The upgrade module contains a Python daemon and a Cronjob.
The Python daemon communicate with backend by subscribing AWS IoT, and receive upgrade message from backend, then notify Cronjob to process.
The Cronjob is responsible for performing upgrade task. Download and extract upgrade package to hidden partition, sync current configuration, add GRUB menuentry and reboot.
Finally complete the system upgrade.

## 说下OS迁移怎么做的
早期客户的虚拟设备OS是CentOS7, 但是CentOS7在2024年停止维护, 为了保证系统安全性, 我们需要将客户虚拟设备迁移到新的OS
OS迁移的难点在于, 如何在重新分区过程中保存并同步客户的配置, 实现自动迁移, 从而提高用户体验
我们通过定制了一个initrd镜像解决这个问题, 在initrd环境中执行以下步骤:
* 把升级包和客户配置从硬盘复制到临时内存, 确保客户配置不会丢失
* 对整个硬盘重新分区, 划分两个系统分区
* 把升级包安装到主分区, 恢复客户配置
* 最后, 通过GRUB设置默认启动项为新的OS, 重启设备完成迁移

In early time, customers' virtual appliances were based on CentOS7. However, CentOS reached end of lie on 2024.
To ensure the security of the system, we need to migrate customers' virtual appliances to a new operation systems.
The challenge is how to sync customer's configurations during disk re-partitioning process and implement automatic OS migration,  then improve customer experience.
We solved this problem by customizing an initrd image, which consists of following steps
* Copy upgrade package and customer configurations from disk to temporary file system to ensure customer's configuration are not lost
* Repartition the entire disk and create two system partitions, then install upgrade package on primary partition and recover customer's configuration,
* Finally, use GRUB to set default boot option to new OS, then reboot virtual appliance to complete migration.

## 回滚是怎么做的
回滚是基于双分区升级方案实现的, 实现比较容易。
客户在CLI上执行rollback命令时, 调用我们写的脚本。 这个脚本使用grubby工具把默认启动项设置为前一个系统分区
设置完启动项后, 再执行reboot, 这样设备重启后就回滚到了上一个分区

The rollback feature is implemented based on our dual-system upgrade solution, so it's easy to implement.
Customer execute rollback command in CLI, and it calls a Shell script we developed. This script used grubby tool to set default boot option to previous system partition. 
Then reboot the virtual appliance, it will rollback to previous partition successfully.

## 说下微服务的集成方案怎么做的
早期版本中, 微服务是集成在固件中的, 这导致虚拟设备ISO镜像体积很大, 有5G左右, 客户安装和升级速度慢, 体验较差
我们的优化方案是, 把微服务的镜像和配置从固件中解耦, 支持客户灵活的在线安装或卸载某个微服务
难点在于:
* 如何避免微服务间的资源竞争
* 如何避免端口冲突
* 微服务如何感知用户的配置变更

做法是:
* 给每个微服务分配一个独一无二的serviceCode, 服务网关根据serviceCode创建同名的namespace, 实现了不同微服务之间的资源隔离
* 每个微服务需提供一个tar.gz部署包, 包括容器镜像和部署yaml文件两部分, 微服务团队负责把部署包发布到AWS S3 
* 每个微服务创建一个configMap, 用户修改配置后, 后端通过消息队列通知虚拟设备, 虚拟设备上的升级模块负责把配置写入configMap 
* 对于微服务依赖的全局配置, 我们创建了一个appliance-configmap, 定时把这个configmap同步给各个微服务, 各微服务监控configMap配置, 获得最新配置
* 对于端口冲突问题, 启用了Microk8s自带的Ingress插件, 所有http请求从宿主机的80或443端口进入。微服务的提供者需要在部署包中定义ingress yaml, 指定path和backend

In early versions, microservices were integrated directly into the firmware. This caused virtual appliance ISO image very large, 
the installation speed is very slow for customers, which negatively impacted user experience.

Our goal is to decouple the microservice image from the firmware, allowing customers to install or uninstall specific microservices flexibly.
There are many challenges, for example, how to avoid resource competition and port conflicts between microservices. How microservices detect user configurations change.

The approaches we took are as follows:
* Assigned a unique serviceCode to each microservice. Service Gateway will create a namespace with the same name as the serviceCode, to avoid resource competition between different microservices.
* Each microservice should provide a deployment package, which includes two parts: container images and deployment YAML files. The microservice team is responsible for publishing the deployment package to AWS S3.
* Each microservice should create a ConfigMap. When customer modifies configuration, backend notifies virtual appliance via a message queue. 
  The upgrade module on the virtual appliance is responsible for updating configuration into the ConfigMap.
* For global configurations that all microservices share,  we created an appliance-configmap and sync it periodically to all microservices. Each microservice monitors its ConfigMap to get latest configuration.
* To address port conflicts, we used Microk8s Ingress plugin. All microservices provide HTTP service use port 80 or 443 on the host machine. Each Microservice should provide an Ingress YAML in their deployment package, specify the path and backend.


## 项目中最大难点时什么, 如何定位解决的
项目最大难点在于问题的快速定位。 比如客户虚拟设备重启时，可能出现各种复杂的K8S问题或网络问题

我举一个k8s问题定位案例
有一个客户反馈, 他们的虚拟设备重启后无法正常工作了
我通过日志发现Microk8s不工作, 由于apiserver证书有效时间不正确; 但是客户坚称他们的虚拟机时间是正确的, 这使得问题变得复杂

好在项目设计的时候，我就考虑到了定位的问题。 我们记录了每次虚拟设备启动日志, 通过回溯日志, 我发现了一条关键线索: 某次启动中, 客户虚拟机的RTC时间比正确时间快了一个月.
为了进一步验证, 我在本地环境中模拟了类似情况, 发现Microk8s的确在每次重启后, 根据当前虚拟机系统时间刷新apiserver证书。 此时, 我终于确认了问题的root cause.

我和客户沟通, 把日志截图提供给客户, 说明了客户的环境问题，建议客户重启解决问题。 
后面我们设计了一个workaround, 通过添加一个定时任务, 检测证书是否有效, 如果证书失效, 就直接刷新证书, 避免了因为客户环境问题导致的服务中断, 提升了用户体验 

The biggest challenge in the project is trouble-shooting. We need to address various complex Kubernetes or network problems.

Here is an example of a K8S issue.

A customer reported that their virtual device failed to work after reboot.
I found that Microk8s didn't work due to invalid cert time of apiserver, but customer insisted that their system time was right. This made the problem become more complex.

Fortunately, I had already considered the trouble-shooting before. We recorded all startup logs. 
By reviewing these logs, I found that during one particular startup, the system time on the customer's virtual appliance was one month ahead of the correct time.

For further validation, I tested in my local machine, and discovered that Microk8s will refresh apiserver cert based on system time when reboot. Finally, I found the root cause.  

I communicated with customer, provided them with logs and explained the root cause. I recommended our customer to reboot and solve the problem.
Besides, I designed a workaround by adding a cronjob to check certificate continously. if cert time is invalid, it would be refreshed immediately.
This workaround can improve user experience.


我再举一个网络问题定位案例

在本地测试过程中，我们遇到了一个问题: 如果虚拟设备配置了双网卡，在升级后有概率出现网络不通的情况。 然而，升级后的网卡IP和Gateway看上去是正确的。

首先，我尝试通过ping网关来验证网络连通性, 这是基本排查步骤。 如果网关不可达, 说明问题可能出现网络接口或路由配置上。 结果发现ping失败
接着, 我通过日志详细对比升级前后的网卡配置变化, 发现升级后网卡的MAC地址有问题, eth0和eth1的MAC地址反了
我进一步搜索资料, 发现问题的根本原因在于, 现代Linux系统通常使用基于硬件信息(如PCI总线ID)的可预测网卡命名规则, 保证迁移后网卡顺序也是正确的
然而我们的虚拟设备采用的还是传统的eth命名规则, 这种规则不能保证网卡顺序的正确性

为了解决这个问题, 我写了一段脚本, 升级前使用ethtool获取两个网卡的businfo, 升级后根据businfo判断, 如果网卡顺序不正确，就通过modprobe重新加载网卡驱动, 最终解决了问题

Here is an example of a Network issue.
During local testing, I found a problem
If a virtual appliance was configured with dual network interface, there may be a network failure after an upgrade. 
However, the IP address and Gateway of two NICs seemed correct after the upgrade. This made the problem more complex.

First, I attempted to ping gateway to test network. Because if the gateway is unreachable, it indicates problemes in NIC configs and routing table. 
Next, I reviewed logs and compared the NIC configurations before and after the upgrade. I found that MAC address of eth0 and eth1 were swapped.

After further research, I found the root cause. Modern Linux systems use predictable network interface naming rules to ensure consistent NIC ordering
However, our virtual appliance still used the traditional ethX naming rule. This may caused wrong NIC order.

To address this issue, I wrote a script. Before the upgrade, I used ethtool to record the businfo of both NICs.
After the upgrade, the script checked the businfo to determine whether the NIC order was correct. 
If the order was incorrect, the script used modprobe to reload the NIC drivers in the correct sequence.

And this solution eventualy resolved the problem.

# 常见项目问题

## 这个项目都做了哪些测试？
黑盒测试, 针对功能点的测试 [TODO]

## 还有什么可以优化的
* 压缩虚拟设备镜像大小
* 精简微服务基础镜像

## 用什么技术开发的, 你做了哪些部分
在服务网关项目中, 我负责虚拟设备的特性开发工作.
* 定制了一个ISO镜像, 镜像包括一个最小的Rocky, Microk8s, 升级模块, 一个CLI命令行
* 开发了一个升级模块, 用于实现虚拟设备固件的升级. 
	* 升级模块包括一个运行在宿主机的Python进程, 通过HTTP和AWS IoT和后端通信, 获取升级包, 执行升级任务
	* 以及一个cronjob脚本, 实现固件的双系统分区升级和回滚. 
* 设计了微服务在虚拟设备上的集成方案, 给本地趋势产品提供服务, 减少客户带宽消耗. 使用k8s, nginx ingress技术

## 几个人开发的, 你怎么合作的
服务网关这个项目有3个核心开发人员, 1-2个QA, OPS
我负责虚拟设备的特性开发工作; 1名开发负责后端(NodeJS); 另1个开发负责前端(React) 
项目中涉及到很多合作
* 和后端的合作, 比如设计后端给虚拟设备提供的AWS IoT消息和Restful API
* 和前端的合作, 比如Forward Proxy微服务中, 如何配置白名单
* 和QA的合作, 实现一个feature或者修复一个bug, 告诉QA修改了什么, 影响有哪些, 如何测试
* 和TS的合作, 处理客户case, 和TS沟通客户问题, 给出解决方案 
* 跨部门的合作, 比方说有的部门部署微服务失败, 帮忙到环境定位.

## 这个项目用了什么第三方软件/插件？
虚拟设备开发用到了一些三方软件, 比如:
* Python升级模块使用了AWS SDK实现MQ消费端, 使用kubernetes库操作k8s
* 使用Microk8s部署微服务, Nginx Ingress插件简化微服务的集成
* 命令行是基于Cisco的开源CLI框架做的
* 正向代理用了Squid, Stunnel开源软件
* 后端APP用了NodeJS Express, ORM库用了Sequelize, 前端用了React

## 这个项目采用了什么样的软件开发流程？
* 两周为一个Sprint版本迭代, 明确交付需求. 每周例行团队会议, 同步进度
* Jira和GitHub issues管理任务, 代码review
* 持续集成与交付, Github Actions作为CI/CD工具


## FPS项目中最大的难题是什么? 如何解决的？**
[TODO]
最大难点就是解决客户网络不通问题。 因为客户网络是一个高度不确定的环境, 绝大多数的网络不通问题都是客户防火墙导致的

防火墙配置错误, 导致FQDN不通过
监听了HTTPS请求, 做中间人, 导致证书校验不通过

HTTPS流程, HTTP代理+HTTPS流程, Squid+Stunnel流程

HTTP CONNECT通道
* 客户端和代理服务器进行三次握手, 建立TCP连接
* 客户端发送HTTP CONNECT请求给代理, 告诉代理目标站点地址和端口号
* 代理收到请求后, 和目标服务器也进行三次握手, 建立TCP连接
* 连接建立后返回200 OK, 告诉客户端加密通道已生成
* 之后代理只是来回转发客户端和服务器间的加密数据包 （TLS握手+HTTP数据)

https://www.hawu.me/operation/886
Stunnel加密通信原理
Stunnel Client部署在防火墙内, Stunnel Server部署在云上(客户防火墙外)
Squid把包发给Stunnel Client
Stunnel Client将包加密, 发送给Stunnel Server
Stunnel Server解密后转发请求
由于通过客户防火墙的数据是被Stunnel加密的, 客户防火墙只能看到Stunnel Server:443的TCP包, 看不到具体的FQDN


写一个KB，让客户先排查是否防火墙配置问题
* 首先检查网络设置和网络连接: ip, netmask, gateway, dns, ping gateway
* 让客户检查防火墙规则: 协议，方向, 行为, 规则顺序
* 禁用防火墙的HTTPS证书替换, 某些防火墙为了检测HTTPS流量, 会做中间人拦截HTTPS流量。 防火墙生成一个新的SSL证书，冒充目标网站证书

举例: 某些客户防火墙为了检测HTTPS流量, 会做中间人拦截HTTPS流量。 防火墙生成一个新证书，冒充目标网站证书, 因为虚拟设备不支持客户自定义CA，导致网络不通
解决方法: CLI中实现诊断网络的命令，通过查看curl的结果, 看issuer是否为entrust, Amazon; 如果issuer是一个IP,hostname或防火墙产商名字，说明被替换了


# 项目2: Forward Proxy Service

## 介绍下Forward Proxy这个项目
这是一个安装在Service Gateway上的微服务, 为本地的趋势终端产品接入云服务提供正向代理功能, 实现本地产品访问互联网的集中管理  

早期, 各个本地产品从云端下载数据, 需要各自访问不同的后端, 只能各自实现一套认证，访问控制方案.
通过引入Forward Proxy服务，所有本地产品连接到这个正向代理即可，实现了集中的认证, 访问控制, 降低了各产品开发和维护成本

通过Microk8s部署, 基于hostNetwork模式, 直接使用宿主机网络栈，监听8080端口，提升性能
核心组件有两部分, 一个是Squid软件, 另一个是Python进程, 感知网络设置和白名单变化, 更新Squid配置文件，重新加载Squid


## 说一下Stunnel的加密通信方案
Forward Proxy安装在客户本地网络中，客户网络是一个高度不确定的环境。 部分客户配置防火墙规则时会出现一些问题, 比如:
* 不支持通配符FQDN
* 配置错误, 或遗漏某个FQDN
* 客户重新配置防火墙规则需要一定时间, 这段时间服务无法工作, 影响用户体验

有两个问题:
* 1. 减少客户的防火墙配置错误导致的网络不通问题 
* 2. 某些特定的FQDN由于性能原因不适合加密, 需要直出或者走客户代理, 其余FQDN数据需要加密走别的代理出。 需要支持这种多代理转发场景

我们参考了科学上网的方案, 基于Stunnel对Squid数据做TLS加密, 从而将终端到云端访问整合为一个FQDN, 简化客户防火墙配置

Stunnel工作原理是:
* 基于TLS加密隧道工具, 客户将未加密流量发到Squid, Squid发送到Stunnel
* Stunnel client端使用TLS加密, 传到Stunnel Server, 防火墙只能看到TCP/443的包，只需要允许Stunnel Server的1个FQDN通过即可
* Stunnel Server解密，把HTTP请求送给目标服务


client -> server
client -> Squid -> server
client -> Squid -> Stunnel -> server 


# 项目3: Matrix仿真平台

## 驱动接口仿真是怎么回事
* 在真实的交换机上，我们领域的业务代码依赖驱动提供的接口
* 要实现仿真环境的验证，需要对驱动接口进行打桩的工作
* 测试人员需要构造故障, 比如通过改变驱动行为的方式。 如果每次都修改C代码再重新编译, 这种测试体验会很差

## 为什么用Redis不用MySQL
* Redis是基于内存的，性能很高。 我这个项目中的数据规模在2w个key左右，QPS最高在1w左右，因此我们直接使用Redis做数据库
* Redis的key-value模型很适合模拟驱动接口行为，灵活方便, 因为我们的驱动接口就是get/set; MySQL是关系型的，还要先建表, 不灵活
* Redis数据类型丰富（string, list, set, zset, hash)，mamcached只支持string, mongodb是文档型数据库，更适合文档存储
* Redis支持发布订阅和数据库通知 。我在项目中使用这两个特性，解决了仿真环境中单板插拔流程的验证问题

## 读写Redis库怎么做的
基于Hiredis这个C语言实现的开源库, 开了一组读写Redis key的API. 选用HiRedis是因为我们被测代码也是so库, 集成很方便 
举例： 把1号单板设置成不在位，通过修改Redis key: "/board#1/present_status"的值实现，无需修改驱动代码

## Redis挂了怎么办, 可靠性怎么考虑
每个单板1个容器，作为redis client; 1个管理容器，安装redis数据库
如果Redis挂了，管理容器会检测到异常，并依次重启管理容器和各个单板容器。
由于这是仿真环境大部分驱动数据是易失性的，无需持久化或集群支持; 对于需要持久化数据, 通过写文件和Docker Volume方式存储

## 利用Redis的键空间通知机制, 开发了一组发布订阅的API
设备管理的业务依赖驱动接口注册一个回调, 从而感知单板插板事件
我们需要仿真驱动接口，保证发生插拔事件可以触发这个回调, 实现单板插板特性在仿真环境的而测试
如果是轮询的方式，定时检查Redis中单板在位状态, 这种方式浪费CPU.

**如何解决的？成果：**
利用Redis的键空间通知机制解决这个问题:
在驱动函数中, 向Redis发起频道订阅, 频道的名称可以是"Board1/present_status", 订阅单板的在位状态
然后模拟单板插入动作，更新Redis键, 触发Redis键空间通知机制。 Redis将消息发布到所有在线的消费者(也就是我们的上层业务)。 
上层业务收到订阅消息后，触发一次业务回调，从而感知到了单板插板。

## 3. 把Redis短连接优化为长连接怎么回事
早期的仿真平台, 驱动接口是通过Redis短连接实现的，导致频繁地建立连接和关闭连接，性能低下。 
设备管理有一段子卡初始化代码，需要查询很多属性, 用短连接耗时很长

**怎么解决的**
* 通过netstat发现TIME_WAIT过多，都是6379端口，判断使用Redis短连接
* 把短连接改造为长连接。 每个进程分一个Redis长连接，各线程通过pthread互斥锁获取长连接
* 将驱动接口平均一次读写时间缩短为原先的1/15，从平均2ms到0.1ms一次，获得显著性能提升。

## 工具是你一个做的？还是合作的？怎么合作的？
这个工具是合作完成的
早期设计阶段, 有1个架构师, 1个SE, 我参与项目设计和方案讨论, 梳理业务做原型验证，和架构,SE讨论方案。
连我在内, 有两个开发，另一个开发来自杭州。 我负责驱动接口仿真和Redis，另一个同事负责Docker镜像, 仿真设备包构建

后期的性能优化也是合作的，连我在内有3名开发，在同一个部门的业务组。
我做方案设计, 梳理流程拓扑, 把开发任务分解到2名开发同事去做。

## 你觉得还有什么优化
* 优化内存。比如对字符串键做压缩存储（因为Redis字符串键的三种编码: long, embstr,raw）
* 优化请求速度，有些驱动函数会一次性读写多个key，用mset, mget命令代替get, set, 减少客户端和Redis的通信次数

## 举一个问题定位案例
当时遇到一个Redis key被意外删除的问题 

我们通过常规的查看日志, tcpdump抓包方法, 没有找到原因.
我当时看过REDIS的单机数据库实现, 想到既然Redis是基于内存的数据库，字符串键肯定在进程中的某个内存地址处，key丢失时这个地址的内容肯定被改写
所以我只需要GDB打个断点，看下调用栈不就搞定了
后来发现是REDIS配置项maxmemory过低, 只给了1M, 导致Redis认为空间不够就随机淘汰了一些key, 修改了这个配置项后问题得到了解决









## 其他项目问题

### 如何压缩全量升级包
* 使用Rocky官方minimal的镜像做定制, 这个镜像在1.7G左右
* 只分配必要空间给构建镜像的虚拟机, 等客户安装成功后再分配剩余磁盘空间. 进一步压缩OVA大小
* 导出OVA前, 清理临时文件，日志文件, 临时关闭swap分区
* 选择Microk8s, 一个轻量化的K8s发行版, 且支持按需加载k8s插件, 进一步节省空间
* 使用XZ压缩算法减小包的体积
最终导出的OVA在2.5G左右

### 在线升级遇到网络不稳定怎么处理的
* 升级时, 虚拟设备从云端获取升级包下载链接和sha256sum校验值。 使用wget -c下载, 支持断点续传。
* 下载完成后, 通过sha256sum校验文件完整性。 如果校验失败，说明下载失败, 此时通知后端下载失败, 提示客户重试。

## 你提到多线程来处理不同类型的任务，但如何保证线程间的同步和数据一致性
使用Python的Queue处理不同类型任务, 利用Queue的线程安全特性, 保证多个线程同时访问队列不会出现数据竞争问题

## 为什么引入消息队列
因为升级任务耗时较长, HTTP交互不适合等待长时间任务. 引入消息队列将后端与虚拟设备解耦，后端只需关注和MQ通信

## 你如何保证消息传输可靠性?

## 你如何防止重复消费
* 首先每个消息都有一个UUID类型的taskID
* 在Python daemon中, 利用Python过期字典，每次消费时, 先获取锁, 检查taskID是否在过期字典中; 如果ID存在说明重复消息; 否则消费消息, 把taskID添加到过期字典
* 第二个是消费消息做到幂等性, 比如升级场景, 如果判断当前版本和目标版本一致, 就不做处理. 保证同一个task执行多次，结果也是一致的。  

## 如何保证消息传递可靠性, 是否有重试机制或消息确认机制 ?
* Aws IoT协议的Qos(服务质量)机制, 支持QOS 1, 至少一次消息传递, 重复发送直到收到PUBACK
* 发送端发送失败后重试3次

## 为什么选择AWS IoT, 有没有考虑其他MQ解决方案 ?
* AWS IoT支持基于Topic的主题分发, 每个虚拟设备通过订阅topic执行升级任务, 功能上可以满足要求
* AWS IoT支持MQTT协议，轻量高效，资源消耗低, 适合本地设备的通信
* AWS IoT是Amazon提供的服务, 经过了企业生产环境验证, 具有稳定性和可靠性, 且我们公司在普遍使用AWS的服务, 选择AWS IoT可以简化开发流程
* 不选择Kafka，是因为我们的并发需求不高, 6k客户,1w+虚拟设备, 主要是低频但关键的任务。 使用Kafka没有明显优势，反而增加复杂性
* 不选择RabbitMQ, 因为这需要自行搭建，维护集群. 增加开发维护成本

## 为什么设计Cronjob，不直接在Python服务里处理升级
这样设计是考虑到职责分离; Python Daemon专注于消息消费和记录任务, Cronjob负责具体任务执行, 这样便于维护和扩展。

## 如何检测微服务运行状态
后端通过消息队列定时发送心跳给虚拟设备, 虚拟设备订阅消息后, 通过kubenetes API查询某个service的Pod是否Running, 再通过POST请求响应给后端

## 启动项切换到备用分区的具体实现方式是什么？
使用grubby管理启动项, grubby是一个专门用于管理GRUB的工具，比手动编辑GRUB配置文件更安全高效
实现步骤概括: 划分独立的boot分区, 禁用os_prober, 使用grubby添加启动项, 设置默认启动项, 重装GRUB
理解GRUB工作原理, Linux启动流程, 完成开发和调试工作。 

## 说下Linux启动流程
https://handerfly.github.io/linux/2019/04/02/Linux%E5%BC%80%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%E8%AF%A6%E8%A7%A3/
* 系统加电, BIOS开机自检
* 按照BIOS设定启动的顺序, 查找可启动设备, 通常是硬盘, 把控制权交给GRUB
* GRUB把内核加载到内存，挂载initrd, 通过initrd加载真正的根文件系统
* 内核启动完成后, 执行第一个用户空间进程init, init负责启动其他服务

## 说下GRUB工作原理
GRUB是Linux的引导加载程序，负责将内核加载到内存中启动，两阶段运行
一阶段位于磁盘的主引导记录(MBR)中，加载二阶段的core.img; core.img读取grub.cfg, 生成启动菜单, 加载指定的内核和initrd

## 为什么选Microk8s, 不选k3s, minukube ?
* 除了Microk8s, 还有minukube, k3s. Minikube只适合本地开发和学习，不是为企业生产环境设计的，缺乏高可用性，排除
* k3s有很多优点, 比如使用二进制文件, 依赖少，资源占用低，社区比Microk8s活跃
* Microk8s的缺点是社区活跃不足, 且安装依赖snapd, 但是这种影响可控。 且项目早期已选择Microk8s,为了保持一致性，减少迁移成本，最终仍然选择了Microk8s

## 如何配置Ingress的? Nginx如何提供统一入口点的，具体实现方式是什么? 具体实现原理是什么?
* 选择了Microk8s默认的Nginx Ingress插件
* 定义一个DaemonSet部署Nginx Ingress Controller, 在DaemonSet中指定hostNetwork, 让Nginx监听宿主机的80和443端口，从而提供外部统一访问入口
* 这种方式避免了NodePort直接暴露端口，从而简化了微服务管理。
* 除了支持HTTP请求，可以通过在DaemonSet定义TCP端口, 支持TCP负载均衡
* 微服务需要提供ingress.yaml, 指定路径路由, 比如把某个路径的请求转发到某个服务。

## 引入Ingress后，如何确保微服务的安全性？
每个虚拟设备注册时，后端会发送公司的证书和私钥, 虚拟设备拿到证书和私钥, 配置在ingress-cert这个secret中

## CLI怎么开发的
虚拟设备安装后，客户需要登录到设备, 做网络配置，进行注册, 才能连到云端。
我们基于Cisco的开源框架实现了一个命令行工具，客户通过CLI完成网络配置，实现注册功能

## 开发了一组诊断命令，用于快速定位 OS、Kubernetes 和网络相关问题
OS: 查cpu, 内存, 进程, disk
网络: IP, 路由, 网卡, iptables, Route
K8S: IMAGE, namespace, pod, configmap, ingress, deployment, services, volume, Microk8s日志
系统日志: 网络没有问题, 可以支持上传日志, 和Remote Shell

## 怎么实现注册的
使用JWT注册方式
* 客户首先在UI上拿到一个由服务平台维护的register_token
* 用户在命令行中输入register_token注册，触发注册请求
* 虚拟设备通过POST请求将CPU,内存,IP信息连同register_token一起发送给后端
* 后端收到请求, 解析请求头的token, 验证签名, 获取customerId
* 校验通过后, 后端为虚拟设备生成一个uuid, 用于唯一标识这台设备, 再生成一个applianceToken
* 后端把虚拟设备信息存到MySQL数据库, 把虚拟设备ID, Token, 还有消息队列的FQDN返回给虚拟设备
* 虚拟设备成功收到响应后，保存applianceId, applianceToken, 用于后续通信   

## JWT工作原理是什么
JWT用于身份认证, 通过签名确保数据完整性和真实性
由三部分组成, Header, Payload, Signature, 工作原理:
* 用户登录成功后，服务器生成一个JWT返回给客户端
* 后续请求中, 客户端将JWT附加到HTTP请求头中, 发送给服务器
* 服务器接收到JWT后, 验证其签名和有效性

优点: 
* 无状态, 服务器无需存储会话, 适合分布式系统
* 跨平台, JWT基于JSON格式, 易于解析使用, 支持多种编程语言

局限性:
* 无法主动失效

## JWT过期时间如何设置的，怎么同步的
每天定时从服务平台同步Token, Token设置了两个月过期时间, 过期前10天，立刻请求服务平台刷新Token


## 并发量有多少，怎么测试的
8vCPU/12G/500G/1000Mbps
1000台agent并发升级, 流量157M*1000, 30分钟下载完毕
CPU 8%, Mem 58%, Disk 1%，峰值内存7G，瓶颈在带宽
157*8/(30*60) = 700Mbps, 相当于1Gpbs网卡

## 为什么选Squid, 是否有其他替代方案(Nginx, HAProxy)?
* Squid作为老牌正向代理, 在生产环境经过验证，稳定可靠.
结合我们的项目需求，正向代理需要支持认证功能, 基于IP,URL的访问控制, 配置用户代理, 以及多代理转发场景(不同的FQDN走不同的代理)
Squid功能丰富，配置简单, 可以容易实现这些功能
Nginx也能支持这些功能, 但是需要通过三方模块, 配置维护复杂一些. 

## Squid正向代理的原理
默认监听3128端口接收客户端连接
Squid使用多进程模型, 主进程不直接处理客户请求, 而是请求分发给worker进程，每个worker进程处理客户端请求 

## Squid的安全认证是如何实现的？ 为什么选择Basic认证, 还有哪些认证方式?
我们使用Basic认证, 请求头中Authorization: 提供用户名和密码
用户名是本地产品名称+guid, 方便密码, 密码设计(用户名+apikey)做SHA1, apikey只有客户知道, 30天过期
代理是HTTP的, 由于虚拟设备在内部网络环境, 就采用了Basic认证方式, 简单快速

## 为什么Squid代理使用HTTP, 不是HTTPS
如果使用HTTPS, 就需要为每个代理服务器生成SSL证书
虚拟设备安装在内网环境， 内网环境使用HTTP代理即可, 证书管理存在维护复杂性
HTTPS涉及加密, 对性能有影响

## 访问控制的具体实现方式是什么？例如是否基于IP地址、或URL 白名单/黑名单？
通过Squid配置文件中定义ACL, 实现访问控制。 
基于FQDN的访问控制, 预设一个白名单, 只允许白名单中的FQDN通过
允许客户在UI上添加白名单, 白名单信息由后端下发到虚拟设备，保存到ConfigMap
Pod中把ConfigMap挂到文件, 如白名单发生变化，从ConfigMap读取新的白名单，再reconfigure

## 配置用户代理作用是什么，怎么做的
客户出于他的网络管理要求，希望所有虚拟设备流量通过他的代理服务器; Squid支持父级代理, 通过配置cache_peer定义父级代理的地址

## Squid 的缓存机制是否被启用？如果有，具体的缓存策略是什么？
没有手动启动磁盘缓存. Squid默认会启动内存缓存, 根据响应头,请求方法决定是否缓存某个请求
Cache-Control 强缓存, Etag 弱缓存; GET, HEAD可以缓存 
 






**DeployMent**
AWS云, 每个site一个EKS
site: US, EU, SG, AU, IN, JP, UAE
site: US-EAST-1
vpc cidr: 10.131.44.0/22
subnets: 
xdr-app-tgw-private-1a	10.131.45.0/24
xdr-app-tgw-private-1b	10.131.46.0/24
xdr-app-tgw-private-1c	10.131.47.0/24
xdr-app-tgw-public-1a	10.131.44.0/25
xdr-app-tgw-public-1b	10.131.44.128/25
routes: 
rtb-03eb993eb0a0df54b
0.0.0.0/0	nat-0d7fa27334ad10a4a
10.0.0.0/8	tgw-0c4d0ec54e6f330f1
10.131.44.0/22	local

**Cost**
2022-11
* RDS 8500$
* EC2 2500$
* EKS cluster 140$
* Load Balancer 133$
* VPC 150$
* Cloudwatch/S3 300$
* IOT 300$
total 12000$

难点:
* AMI 3.0问题
* whitelist配置变化如何通知到到FPS
* 502 Gateway 问题
* 分析流量

##
## 最大的难点是什么, 你是如何解决的
项目中最大的挑战在于虚拟设备固件升级方案的设计与实现, 难点体现在两个方面:
* 一个是没有先例可循; 公司内部没有部门实现过类似的升级方案, 当时也没有chatgpt, 需要基于对技术原理的理解推导方案
* 另一个是涉及的技术比较多; 比如Linux技术, 需要了解启动流程, 分区管理, initrd定制, ISO定制, GRUB引导程序, 比如K8S, Docker, HTTP, 消息队列, Python等
不仅要求我对整个升级流程有深入了解，还需要整合多种技术实现了一个可靠的解决方案.

为了解决这些难点, 我做了几件事:
* 首先在设计阶段，花了大量时间学习相关技术, 确保对升级流程有了清晰的认识; 和团队沟通, 明确升级方案的具体需求
* 接着是模块化设计，把整个升级流程分为多个独立模块, 比如升级包制作模块, 在线升级模块, 每个模块专注于解决特定问题, 降低开发难度, 提高可维护性
* 然后是技术选型, 选择最合适的工具满足需求, 例如: 使用AWS IoT实现云端到设备通信; 使用轻量化的Microk8s部署服务

目前有6k+企业客户, 1w+台虚拟设备使用我的升级方案, 这套方案降低了开发维护成本, 同时显著改善了用户体验