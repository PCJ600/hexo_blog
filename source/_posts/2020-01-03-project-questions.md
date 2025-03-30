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

## 简单介绍项目
Service Gateway是一个部署在企业网络的服务网关, 采用混合云架构, 将云端服务转移到客户本地环境, 从而减少客户的带宽消耗. 
该服务网关由一个云端的管理平台和一组本地的Rocky Linux虚拟设备组成, 支持VMware, AWS, Azure多平台部署;
并通过Microk8s集群方式部署虚拟设备, 提供了多种XDR关键服务, 支持5万个企业客户, 覆盖1亿台终端


## 1.为On-premises网关设备设计了基于Linux双系统分区的升级方案, 涉及分区设计, 升级包构建, 系统配置同步, GRUB启动项配置等关键环节. 此方案解决了系统故障后无法回滚的问题, 并删除了1GB的增量升级代码, 降低了40%维护成本
早期，客户的On-premises网关设备（基于CentOS 7）采用增量升级方案进行固件更新。 这种方案存在两个主要问题
一旦升级过程中出现问题，客户只能重新安装系统，严重影响用户体验。
部分客户未开启自动升级功能，为了支持所有客户的升级需求，升级包必须保存从初始版本到最新版本的所有增量文件。随着版本迭代，升级包变得越来越大，难以维护。

**方案**
针对这些问题，我设计并实现了一个基于Linux双系统分区的全量升级方案, 像Android系统、 Chrome OS及许多嵌入式和IoT设备都采用双分区系统升级方法, 做到无缝更新和故障恢复.
全量升级方案中包含一些关键技术环节: 设计分区, 构建升级包, 同步系统配置, 配置GRUB启动项。 

设计分区： 使用BIOS+GPT分区格式，兼容现有客户。分区方案包括：BIOS Boot分区、Boot分区和LVM分区, 在LVM分区上创建名为VA的逻辑卷组，并进一步创建四个逻辑卷：VA-root和VA-back作为两个系统分区，VA-data存储公共数据，VA-image存储镜像文件。
构建升级包： 通过ISO安装虚拟机, 导出虚拟机OVA文件, 解压OVA文件得到VMDK磁盘文件, 使用guestmount挂载VMDK文件，对整个根文件系统进行打包，注意打包时需保留UID（numeric-owner）和文件扩展属性（xattr），解包时同样需要声明UID和xattr。

**难点**
目标系统的GRUB版本(OS版本)可能比当前系统更高, 如果用当前系统GRUB引导两个系统分区, 会存在兼容性问题. 针对这种情况，我采取了以下做法
* 使用目标系统的/boot引导两个系统的内核。
* 通过chroot方式配置并重装GRUB。
* 如果GRUB配置失败，回滚到升级前的状态。

为什么不直接把目标系统的/boot放在LVM系统分区？
* 兼容性问题, 传统BIOS和某些UEFI固件不支持直接从LVM卷启动
* 复杂性增加, 故障恢复困难

为什么不划两个/boot分区？
* 手动修改风险：客户需要手动调整磁盘分区，这不仅增加了操作难度，还可能导致数据丢失或分区表损坏。
* 实现复杂性：理论上划分两个/boot分区是可行的，但实现起来较为复杂。实际上，GRUB的一个/boot分区就可以引导多个版本的内核，支持向前兼容。

**效果**
成功解决了系统故障后无法回滚的问题, 极大提升了客户体验. 基于双分区方案实现系统回滚非常容易, 只需配置GRUB默认启动项为上一个系统分区，再重启即可
同时移除了1G多的增量升级代码, 显著降低了维护成本

**遇到的问题**
问题1: 升级后文件权限不对, admin用户无法登录
定位过程：后台查看发现admin用户的文件UID不正确，升级前后两个系统的用户配置不一致。同样的admin用户在两个系统上的UID不同。由于升级包是通过tar备份的，而tar解压文件的默认行为是根据当前系统用户名进行映射，而不是UID，因此升级后文件的UID不正确。
解决方法：在tar解压时添加–numeric-owner参数，保留文件的UID，解决了问题。

问题2: 升级到Rocky Linux 9.4后, 客户设备启动故障, 报错: 虚拟机CPU不支持x86_64_v2
定位过程：查看/proc/cpuinfo，发现客户虚拟机不支持x86_64_v2指令集，但物理服务器支持该指令集。
解决方法：指导客户开启CPU虚拟化扩展，并在升级前验证虚拟机CPU是否支持所需指令集。如果不支持，则直接上报失败。

问题3: 配置了双网卡的客户, 在升级到Rocky Linux 9.4后有概率出现网络不通, 网卡的IP,Gateway配置看上去是正确的
定位过程: 首先ping网关, 发现网关不可达, 通过ethtool查看网卡信息, 发现升级后eth0和eth1顺序反了, 导致网络不通
原因分析: 现代Linux系统通常使用基于硬件信息（如PCI总线ID）的可预测网卡命名规则，保证OS升级后网卡顺序也是正确的。 但是我们的网关设备采用的还是传统的eth命名规则，这种规则不能保证网卡顺序的正确性。
解决方法: 升级后用ethtool判断网卡顺序是否正确, 如果不正确, 用modprobe按正确顺序加载网卡驱动; (如果两个网卡驱动相同, 还需要修改udev规则)


**其他问题**
讲下GRUB工作原理
GRUB（GRand Unified Bootloader), Linux系统中广泛使用的引导加载程序, 负责加载操作系统内核到内存并启动
划分3阶段, Stage 1, Stage 1.5, Stage 2, 其中Stage 2是GRUB的核心部分，包含用户界面、配置文件解析器以及加载内核和initrd的工具。
当选择了启动菜单中的某个条目后, GRUB会将指定的kernel和initrd加载到内存中, 内核被加载到内存后, 控制权转移给内核, 继续启动流程

讲下Linux启动流程
* 系统加电, BIOS开机自检
* 按照BIOS设定启动的顺序, 查找可启动设备, 通常是硬盘, 把控制权交给GRUB
* GRUB把内核加载到内存，挂载initrd, 通过initrd加载真正的根文件系统
* 内核启动完成后, 执行第一个用户空间进程init, init负责启动其他服务
 

## 2. 通过定制initramfs进入紧急模式, 预先将升级包备份到内存后再重建磁盘分区, 实现了从单系统分区到双系统的无缝迁移, 大幅提升了用户体验
**背景**
我们设计了一个Linux双系统分区的升级方案, 用于解决单分区方案中, 升级后出现故障无法回滚的问题。 但是早期客户的虚拟设备只有一个系统分区, 我们需要设计一种方案, 让客户的网关设备从分区无缝迁移到双分区
难点在于实现无感知的升级, 无需客户手动操作(比如添加磁盘, 创建机器, 配置设备).

**方案**
通过定制initrd进入紧急模式，在initrd阶段把升级包和配置文件从磁盘读到内存，重新建立LVM磁盘分区,
再解压升级包到主系统分区, 同步客户配置和口令,
最后重新安装GRUB, 重启后迁移到新系统

**难点**
方案中涉及到一个难点: initrd内存空间是有限的, 客户标准内存配置12G, 最低内存只有8G, 需要尽可能压缩升级包, 使得升级包+解压后文件远小于8G, 否则会因为内存不足迁移失败. 我设计的对策是:
* 使用CentOS 7官方miminal ISO做镜像, 这个镜像只有900M
* 只分配必要磁盘空间, 比如导出虚拟机时, 磁盘只分了8G, 等客户机器迁移成功后再自动分配剩余空间到LVM磁盘
* 导出虚拟机前, 清理临时文件, 日志文件, 禁用交换分区
* 使用压缩比率高的算法, 我们选的xz
最终我们的升级包大小是1.7G, 解压磁盘文件大小是3G左右, 实测运行内存在4-5G左右, 没有出现客户因内存不足导致的迁移失败

**效果**
实现了从单系统分区到双系统的无缝迁移, 避免了客户重装机器, 扩容磁盘, 配置设备, 极大地提升了用户体验

**遇到的问题**
仍然有少量客户(几十个）迁移失败, 比如:
* 磁盘有坏道, 这种只能建议重装
* 客户网络问题, 升级后因为网络原因没有注册, 让客户手动注册解决问题


**如何压缩全量升级包**
* 使用Rocky官方minimal的镜像做定制, 这个镜像在1.7G左右
* 只分配必要空间给镜像, 等客户安装成功后再动态分配剩余空间
* 导出OVA前, 清理临时文件，日志文件, 禁用交换分区
* 使用精简置备（thin provisioning）的虚拟磁盘格式，这仅分配实际使用的存储空间，而非预先分配整个磁盘容量。
* 使用XZ压缩算法
效果: 最终导出的OVA在2.5G左右

## 3. 开发了一个控制模块，通过消息队列实现在线升级功能, 引入心跳检测解决微服务同步的问题; 设计了预下载机制, 用户等待时间减少了90%, 同时提升了升级的稳定性

**背景和方案**
为了实现虚拟设备的自动升级和微服务的安装，我们开发了一个运行于虚拟设备上的控制模块
控制模块包含一个Python后台进程, 该后台进程订阅AWS IoT消息队列, 接收云端的升级指令
当控制模块接收到消息时，会解析消息中的taskType字段, 将消息转发到相应的队列, 由对应的线程执行具体任务

**难点**
这里有个问题: 完成固件升级后, 我们需要把客户在升级前安装的微服务重新装起来，更新到最新版本。
为了让云端知道固件升级是否完成, 我们引入了心跳检测机制, 云端每5分钟就会向消息队列发送一次心跳, 当虚拟设备完成固件升级后，会在心跳响应中带上固件版本号, 
云端收到响应后检查版本号就能确认升级已完成，并下发安装微服务的任务, 完成微服务的同步

**其他问题**

**升级流程具体步骤**
客户在UI上手动升级, 或者定时任务触发升级
云端发送升级任务到消息队列
控制模块订阅消息, 生成一个task_file, 把升级任务交给cronjob处理
cronjob脚本解析task_file, 下载升级包, 执行升级任务

**为什么引入消息队列**
实现解耦通信, 云端只需要把任务发送到消息队列, 无需和每个虚拟设备直接交互, 降低耦合度，提高可维护性
实现异步处理, 云端无需等待任务执行完成, 从而提高系统响应速度

** 为什么选择AWS IoT, 有没有考虑其他MQ解决方案
稳定性和可靠性, AWS IoT由Amazon提供的成熟服务案, 经过了企业生产环境验证, 考虑到我们后端也是基于AWS EKS部署的, 使用AWS IoT实现无缝集成, 简化系统架构
易于部署, AWS IoT是完全托管服务, 不需要部署实例
带宽消耗低, AWS IoT支持MQTT协议，适合资源受限的设备间进行轻量级消息传输
成本低, 费用就是消息传输费(每百万条价格1美元), 所有site每月300美金
  不选择RabbitMQ: 费用较高, 按照实例和存储收费, 实现高可用需要部署多个实例, 使用更高; 有额外维护成本
  不选择Kafka: 对我们场景而言, 没有优势, 我们项目并发量不大, 采用Kafka导致资源过度配置, 花钱, 增加系统复杂性

**消息格式怎么设计**
每个虚拟设备分配一个独一无二的topic, 消息采用JSON格式
设计taskId字段, 唯一表示某个任务, 用于云端跟踪任务的状态
设计taskType字段区分不同类型的任务, 例如固件升级, 微服务安装
设计body字段存放具体的内容

**怎么保证消息不丢失**
生产者角度
  要有重试机制; 比如云端在业务代码中判断超时后重试发送多次
  另外有补偿机制, 发送消息前会将任务记录到数据库, 通过cronjob定时检查未收到响应的任务进行补偿(重试3次, 每次间隔5分钟) 
中间件角度
  开启持久会话(cleanSession=false), 客户端断开链接后消息不丢失
  设置Qos 1(确保至少一次传递), 消息会被保留直到确认或超时
消费端角度
  收到消息后发送ACK给消息队列, 让消息队列及时清除消息

**如何避免消息重复消费**
幂等设计, 无论消息被处理多少次, 结果都是相同的
过期字典, 每次消费时先检查taskId是否在过期字典中; 如果ID存在说明消息重复; 否则消费消息之后，把taskId添加到过期字典。

**为什么说MQTT带宽消耗低**
相比HTTP报文, MQTT报文更小, 消息头部通常只有2字节
MQTT使用持久化链接, 设备只需要一次TCP链接就能持续通信，避免频繁连接和断开带来额外开销

**预下载机制**
升级时候发现一个问题, 有很多用户会在同一个时间设置定时升级, 集中下载造成客户瞬时网络流量大, 增加升级失败概率
为了解决这个问题, 我们设计了预下载机制, 对不同的客户设置了不同的预下载时间窗口，将预下载时间设定为发布前12小时的某一个时间点, 分散了客户下载行为，缓解了网络拥堵, 提高稳定性

**如何判断回滚**
通过心跳检测, 虚拟设备在心跳响应中带上版本号, 云端用这个版本号和数据库中的对比，如果心跳中的版本号低于数据库里的，说明发生了回滚

**网络配置变更怎么重连**
一个线程读取配置，更新到内存
另一个线程读到配置变化, 做重连

**如果升级版本有问题, 怎么最小化用户影响**
设计灰度升级方案, 我们的服务网关部署在全球多个地区(比如日本,欧洲,新加坡)。 为了防止升级失败对所有客户造成影响, 我们首先选择客户数量较少的地区发布, 确认整个地区的升级没有问题，再逐步发布到其他地区。
对于同一个地区, 先指定少量客户升级(通过launchDarklyKey API), 只有这些客户升级成功，再让剩余客户升级 
对于有多台虚拟设备的客户, 也是采用分批升级。 首次只升级一半设备, 等下一个周期后先判断这些设备是否升级成功,如果升级成功再升级剩余一半

**在线升级遇到网络不稳定怎么处理的**
* 升级时, 虚拟设备从云端获取升级包下载链接和sha256sum校验值。 使用wget -c下载, 支持断点续传。
* 下载完成后, 通过sha256sum校验文件完整性。 如果校验失败，说明下载失败, 此时通知后端下载失败, 提示客户重试。

controller线程
process iot task (common, metrics)
process heartbeart
monitor appliance config (从磁盘读最新配置，和内存比较，如果配置有更新就重连)

iot health (请求backend获取连接状态, 如果是disconnect, 同步ntp时间, 如果是unregister, 去注册(补偿))
* 定时call backend, 获取server时间， 如果时间误差在半小时以上，修改系统时间


## 4.优化了微服务的集成方案, 将微服务从虚拟设备固件中解耦, 并使用Microk8s Ingress提供统一的访问入口, 将虚拟设备镜像从4.5G压缩至2.7G, 客户安装时间缩短了40%
**背景和方法**
在早期版本中, 微服务是通过RPM包方式集成在虚拟设备固件中, 导致虚拟设备镜像非常大, 有4.5G左右, 客户安装很慢, 体验很差。
为了解决这个问题, 我们优化了微服务的集成方案, 将微服务从固件中解耦, 做了一些改进措施:
* 比如资源隔离, 给每个微服务分配一个独一无二的serviceCode, 网关设备根据serviceCode创建同名的namespace, 实现了不同微服务之间的资源隔离
* 标准化部署包, 每个微服务需提供一个tar.gz部署包, 内容包括镜像和yaml资源文件两部分, 各个微服务团队把部署包发布到AWS S3
* 对于外部访问集群的问题, 我们使用Microk8s Ingress插件, 所有HTTP请求从宿主机的80和443端口进入, 基于URL路径进行路由, 避免端口冲突问题, 简化了微服务管理

**效果**
虚拟设备镜像从4.5G压缩至2.7G, 客户安装时间缩短了40%m 提升了用户体验

**其他问题**

**为什么选Microk8s, 不选k3s, minukube**
* 除了Microk8s, 还有minukube, k3s. Minikube只适合本地开发和学习，不是为企业生产环境设计的，缺乏高可用性，排除
* k3s有很多优点, 比如使用二进制文件, 依赖少，资源占用低，社区比Microk8s活跃
* Microk8s的缺点是社区活跃不足, 且安装依赖snapd, 但是这种影响可控。 且项目早期已选择Microk8s,为了保持一致性，减少迁移成本，最终仍然选择了Microk8s

**微服务更新如何实现的**
虚拟设备通过云端下载部署包, 重新创建namespace, 导入新的yaml资源文件和镜像
实现简单, 但是会有服务中断

**为啥选Ingress**
Microk8s自带Ingress插件，配置和部署都非常简便, NodePort类型每个服务都要暴露一个端口, 增加管理复杂度, 可能导致端口冲突

**Ingress原理**
Ingress解决问题是从集群外部如何访问集群内的某个服务
它的原理是, 在Microk8s里运行了一个Nginx Ingress Controller, 负责监听k8s中的Ingress资源，当检测到Ingress资源更新时, 动态更新Nginx配置文件
外部流量先到达Nginx, 再基于域名和URL将请求转发到Service, 最终把请求转发到具体的Pod

**SSL终止**
SSL终止（SSL Termination） 是指在负载均衡器、代理服务器或入口网关（如使用Microk8s Ingress时）上处理并解密传入的HTTPS请求，
而不是让这些请求直接到达后端服务。这意味着加密和解密的工作由前端设备完成，而后端服务接收到的是已经解密的HTTP请求。
虚拟设备注册时，云端会发送证书和私钥, 虚拟设备拿到证书和私钥, 配置在ingress-cert这个secret中

**微服务之间如何通信的**
通过k8s内置的服务发现机制, 微服务间通过服务名称调用

**如果需要回滚怎么办**
没有自动的回滚机制，需要客户retry
优化: 使用helm自动回滚

**还有什么优化的**
通过helm进行管理, 每个微服务创建一个helm Chart并发布到仓库, 通过helm upgrade进行升级

## 5. 基于CISCO CLI开发了一组命令, 实现了配置管理功能; 通过设计诊断命令, 收集日志, 显著提升了客户本地环境中定位Microk8s问题的效率
**背景**
客户安装虚拟设备后, 需要做必要的配置才可以连到云端, 比如配置网络; 所以我们设计了一个CLI, 客户通过CLI完成虚拟设备的配置
同时, 为了提高问题定位效率, 我们设计目标是, 问题都通过CLI命令或者分析日志文件解决, 尽可能不要远程会话

客户的虚拟设备安装在本地网络, 遇到问题定位后很麻烦, 目标是让问题通过CLI或者分析日志就能解决，不需要远程remote session, 提高解决问题效率

**配置功能包括哪些**
配置命令, 比如配置网络(IP, GATEWAY, DNS, 代理, 主机名), 配置时间NTP, 功能包括注册, 重启, 回滚

**诊断命令设计**
诊断网络问题:
查看网络配置, 比如IP, 网关, 路由表, 代理, iptables规则
网络测试, ping, curl, nslookup

诊断K8s问题:
journactl日志
microk8s inspect
查看资源, 比如pod, configmap, service

系统日志
CPU, 内存, 磁盘, Network, 进程, 定时任务, 系统启动日志等

**怎么实现注册的**
使用JWT注册方式
* 客户在UI上拿到一个由服务平台维护的register_token
* 用户在命令行中输入register_token注册，触发注册请求
* 虚拟设备通过POST请求将CPU,内存,IP信息连同register_token一起发送给后端
* 后端收到请求, 解析请求头的token, 验证token, 获取customerId
* 校验通过后, 后端为虚拟设备生成一个uuid, 用于唯一标识这台设备, 再生成一个applianceToken
* 后端把虚拟设备信息存到MySQL数据库, 把虚拟设备ID, Token, 还有消息队列的FQDN返回给虚拟设备
* 虚拟设备成功收到响应后，保存applianceId, applianceToken, 用于后续通信   
每天定时从服务平台同步Token, Token设置了两个月过期时间, 过期前10天，立刻请求服务平台刷新Token

**JWT工作原理是什么**
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

**CLI怎么开发的**
虚拟设备安装后，客户需要登录到设备, 做网络配置，进行注册, 才能连到云端。
我们基于Cisco的开源框架实现了一个命令行工具，客户通过CLI完成网络配置，实现注册功能

**开发了一组诊断命令，用于快速定位 OS、Kubernetes 和网络相关问题**
OS: 查cpu, 内存, 进程, disk
网络: IP, 路由, 网卡, iptables, Route
K8S: IMAGE, namespace, pod, configmap, ingress, deployment, services, volume, Microk8s日志
系统日志: 网络没有问题, 可以支持上传日志, 和Remote Shell

**问题定位案例**
我们遇到一个客户反馈他们的虚拟设备重启后无法正常工作。经过日志分析发现，Microk8s的apiserver证书有效时间不正确，导致服务无法启动。尽管客户坚称虚拟机时间准确，但问题依旧存在。

为了定位问题，我们回溯了每次虚拟设备启动的日志，发现某次启动时虚拟机的RTC时间比实际时间快了一个月。
为了验证这一假设，我们在本地环境中模拟了类似情况，确认Microk8s在每次重启后会根据当前虚拟机系统时间刷新apiserver证书，从而导致证书失效。最终确认问题是由于宿主机RTC时间错误引起的。

与客户沟通时，我们提供了相关的日志截图，明确指出其环境中的RTC时间问题，并建议他们重启虚拟机以同步正确的时间。
为了避免类似问题再次发生，我们设计了一个定时任务来检测apiserver证书的有效性。如果证书失效，则自动刷新证书，确保服务不间断运行。
这样不仅解决了客户的即时问题，还通过自动化的机制提升了系统的稳定性和用户体验。


## 其他问题
**怎么合作的**

**项目还有什么可以优化的**
* 优化固件升级方案, 使用EFI系统分区，为每个系统创建独立目录存放引导文件, 避免直接覆盖/boot分区可能导致旧系统的引导文件丢失
* 优化微服务的更新流程, 使用helm部署微服务, 支持滚动更新
* 精简微服务基础镜像, 减少服务安装时间, 使用轻量级基础镜像, 多阶段构建, 移除不必要文件等手段, 进一步压缩微服务镜像, 加快安装速度

**用什么技术开发的, 你做了哪些部分**
在On-premises网关设备的升级方案中，我设计并实现了基于Linux双系统分区的架构，解决了系统故障后无法回滚的问题。 涉及Linux启动, 分区, 升级相关技术

使用Python开发了一个控制模块，利用AWS IoT消息队列实现在线升级功能，并通过心跳检测机制解决了微服务之间的同步问题。

将微服务从虚拟设备固件中解耦，使用Microk8s Ingress作为统一访问入口，成功将虚拟设备镜像从4.5G压缩至2.7G，客户安装时间缩短了40%。

基于CISCO CLI开发了一组命令，用于配置管理和问题诊断，通过日志收集和分析工具，显著提升了客户在本地环境中定位Microk8s和网络问题的效率

**这个项目采用了什么样的软件开发流程**
* 两周为一个Sprint版本迭代, 明确交付需求. 每周例行团队会议, 同步进度
* Jira和GitHub issues管理任务, 代码review
* 持续集成与交付, Github Actions作为CI/CD工具

# 项目2: Forward Proxy Service
Forward Proxy Service是一个部署在服务网关上的正向代理, 为客户终端上云提供统一出口. 该服务解决了客户在防火墙策略配置上的痛点, 同时实现了一个Air Gap（终端网络隔离, 代理上云）的解决方案. Forward Proxy单个服务实例可支持高达3万个并发连接

**背景**
解决客户痛点:
客户必须在自己的防火墙上手动配置大量的云端服务URL, 使得本地终端产品能够连接到云端服务; 由于不同产品需要连接到不同的云端服务URL, 并且客户环境中运行了大量的终端, 让客户为这些终端逐一指定防火墙策略是非常困难的
解决方案:
通过FPS, 所有本地终端产品可以通过单一代理服务访问后端服务, 客户只需要在防火墙上配置允许通过FPS的URL规则, 无需为每个本地终端单独设置规则, 显著简化了防火墙策略配置工作量

**Air Gap**
对于银行这种敏感的客户, 其内部网络与外部网络之间是隔离的, 即所谓的Air Gap环境, 这种情况终端无法直接连接到云端服务。 
为了让这些终端可以连到云端服务，设计了Forward Proxy Service解决方案, 
客户把代理部署在DMZ区域, 所有终端设备的出站请求都经过这个代理进行转发, 通过代理上云.

### 基于Squid实现了正向代理, 支持Basic认证和白名单配置, 通过ACL限制终端访问, 显著提升了系统安全性

**如何实现Basic认证**
Basic认证需要终端提供用户名和密码
用户名基于终端产品的GUID, 产品名称, 主机名生成，使用SHA1算法, 对用户名+APIKEY进行哈希之后生成密码
每个客户有一个全局APIKEY, 就是UUID
客户 可以通过UI重新生成APIKEY
新的APIKEY在30天内与旧的APIKEY共存，确保平滑过度

将configmap挂载到磁盘, 通过一个线程定时读取文件与内存中配置比较，如果检测到变化, 就把新配置更新到内存, 再通过队列通知另一个线程处理
另一个线程从队列中消费, 读取内存中配置, 判断apikey变化后写到YAML文件

**除了Basic认证, 还有哪些认证方式, 为什么选择Basic认证**
简单高校, 在企业内部网络安全性没有问题

Digest认证(摘要认证), 对Basic认证一种改进, 通过哈希算法对user, passwd和随机数加密, 避免明文传输密码
Token认证, 请求头中携带一个token, 服务器通过验证token有效性确认用户身份
Kerberos认证, 基于票据认证协议, 配置复杂

**白名单机制的背景是什么**
比如说客户某个终端感染了恶意软件，试图访问外部的恶意网站，白名单机制会直接阻止这些请求，防止客户防火墙没有正确配置导致的可疑请求泄露

**白名单配置如何实现的**
利用Squid的ACL(Access Control List)功能实现了基于URL的访问控制, 限制终端设备只能访问授权的云端资源
我们预设了一个FQDN白名单, 支持域名和IP, 域名支持开头通配符，比如`*.trendmicro.com`
客户可以通过UI添加或修改白名单, 白名单规则存储在ConfigMap, 通过挂载方式映射到Pod的文件系统中
我们在Pod中做了一个agent进程, 定期检查白名单配置, 检测到白名单更新时, 修改Squid配置文件, 重新加载Squid

**为什么选择Squid, 有没有其他方案**
选择Squid, 是综合考虑开发难度, 成熟度, 社区支持, 功能需求
Squid是一个老牌的正向代理, 在企业场景中经过了长期验证, 成熟稳定, 社区支持好
Squid支持正向代理、基于 URL 的访问控制、Basic 认证, 配置简单, 无需插件
Squid能够支持数万的并发连接, 符合我们需要
没有选Nginx, 因为需要额外模块实现正向代理, ACL, 配置较复杂

**单个服务实例支持高达3万个并发连接, 如何保证这种并发能力的**
事件驱动模型: 通过异步非阻塞方式处理多个连接, 基于epoll和非阻塞IO, 一个进程处理上万连接
多进程架构： 通过多进程架构进一步并发能力。 主进程监听客户端连接, 将连接分发给多个工作进程, 充分利用多CPU计算能力
内存优化: 使用内存池减少频繁内存分配释放, 对HTTP响应进行缓存

**惊群现象是什么，怎么保证每个新连接只会一个工作进程处理**
多个进程都在等待同一个事件, 一旦事件发生, 所有进程都会唤醒, 但实际上只有一个进程能成功处理该事件, 其他进程则会阻塞或重新等待, 导致不必要的性能损失
Squid启用SO_REUSEPORT, 由内核自动将新连接分配给监听同一个端口的不同进程

Nginx中通过accept_mutex防止惊群现象, 每个工作进程都有机会获得锁, 一旦某个工作进程持有锁，其他进程不会尝试接收新连接, 知道进程释放锁
Nginx启动时分配一块共享内存区, 工作进程都可以访问这块内存，工作进程通过原子操作CAS获得锁


### 深入分析并迅速解决客户网络不通的问题, 涉及HTTP隧道和TLS握手原理

**为啥代理不用HTTPS的呢**
因为虚拟设备是临时性的，随时会被客户重装，如果用HTTPS, 每次部署都要分发一个HTTPS证书, 后期还有更新维护证书, 引入不必要的开发维护成本
HTTPS加密解密有性能消耗, 这个性能消耗是不必要的

**说一下HTTP隧道的原理**
HTTP隧道常用于两台网络受限的机器之间建立网络连接。 客户端通过HTTP CONNECT请求与代理建立隧道, 从而访问HTTPS, 流程:
* 客户端和代理服务器三次握手, 建立TCP连接
* 客户端发送HTTP CONNECT请求给代理, 告诉代理自己需要连接的目标服务器
* 代理收到请求后, 和目标服务器建立TCP隧道
* 代理返回200 Connection Established给客户端, 告诉客户端整个隧道已经建立
* 隧道建立后, 代理服务器只负责在客户端和服务间之间转发数据，不解析或修改数据。这保证了 HTTPS 数据的安全性，即使代理也无法解密 TLS 加密的内容（因为代理不知道密钥, 没有服务器的私钥)

**说一下HTTPS的原理**
HTTPS是基于HTTP的安全协议, 通过在HTTP和TCP层加入了TLS协议实现数据加密, TLS握手步骤:
* 客户端发送ClientHello，包含支持的TLS版本, 加密套件, 客户端随机数
* 服务器返回serverHello，选择双方都支持的TLS版本和加密套件，并附带自己的随机数和证书
* 客户端校验服务器证书, 如果校验失败终止连接
* 否则根据选定的加密算法(RSA或DH算法)，客户端和服务端协商一个共享的对称密钥
* 最后双方使用协商出的对称密钥进行加密通信

**详细描述下客户网络不通的问题, 如何利用这些原理解决网络问题的**
难点: 出现网络不通问题时, 可能有多个故障点, 包括终端设备, 虚拟设备, 客户防火墙, 目标服务器等. 客户网络环境是不确定的, 这增加了问题定位难度

我们遇到典型案例包括：
案例: TLS握手失败
客户终端通过Forward Proxy代理访问云端服务时网络不通. 从Squid日志中看到请求状态码为200 OK，但响应数据包大小仅为40多个字节，远小于正常预期值
让终端同事配合复现问题, 在网关设备上用tcpdump抓包, 发现TLS握手过程中, client hello发送了，但是没有收到server hello, 这说明TLS握手存在问题
用wireshark查包内容，发现客户端和目标服务器使用的TLS加密套件不匹配, 导致TLS握手失败

案例: 客户防火墙开启了HTTPS监控, 做了中间人, 导致证书校验不通过
当防火墙作为中间人拦截HTTPS流量时，它会用自己的证书替换原始服务器的证书, 我们网关设备不会信任客户防火墙证书, 所以无法建立连接
我们在CLI上开发了一个诊断命令行, 通过curl打印请求详细信息，如果证书是一个IP或者防火墙厂商，说明客户防火墙做了中间人, 让客户关闭这个功能

案例: 访问特定FQDN时出现超时或返回502 Bad Gateway
原因是客户防火墙配置有问题, 没有放行这个FQDN, 如果丢弃报文就是超时，拒绝就会返回5XX错误

### 设计了一个基于Stunnel的加密通信方案, 显著减少了因客户防火墙配置异常导致的网络问题, 极大提升了用户体验

**背景**
在本地终端访问是云端资源时，涉及到大量的FQDN（例如 *.trendmicro.com）。
然而，一些客户的防火墙比较老旧，不支持基于FQDN的通配符匹配。
这些客户必须手动逐个配置每个FQDN，这不仅繁琐，而且容易遗漏或出错，导致网络被防火墙阻挡，严重影响用户体验。

**方案**
针对这个问题，我们设计了一种基于Stunnel的加密通信方案. Stunnel是一个开源软件，提供TLS加密服务; 我们使用Stunnel绕过客户防火墙的限制。
部署架构: 在客户防火墙内部署Squid + Stunnel Client, 在防火墙外部部署Stunnel Server + Squid, 客户终端将代理设置为防火墙内部的Squid。

由于流量都通过Stunnel加密通道传输，客户不再需要在防火墙山上给逐一指定目标服务器的FQDN，只需允许通往Stunnel服务器443端口的流量即可
避免了防火墙通过明文HTTP CONNECT报文或TLS握手过程中的Client Hello报文中的SNI字段识别目标服务器的问题。

**效果**
极大简化客户防火墙配置, 减少了网络问题

**为什么不能只用Squid？**
如果仅在防火墙外部部署Squid, 防火墙可以通过分析HTTP CONNECT报文识别目标服务器，从而对特定域名进行拦截
TLS握手过程中Client Hello报文中的SNI字段也会暴露目标服务器信息，导致防火墙能够识别并拦截特定域名

**如何协作的**
墙外的Stunnel Server + Squid：由其他团队负责部署和维护，确保外部流量的安全性和稳定性。

**有哪些改进的地方**
尽管Stunnel提供了强大的加密功能，但在高并发场景下可能会出现性能瓶颈。
优化措施是水平扩展：通过增加更多的Stunnel实例（SG）来分散负载，提高系统的整体处理能力。

**Squid日志和监控**
为了确保系统的稳定运行，我们实施了以下日志记录和监控措施：
* 健康检查：使用Kubernetes的livenessProbe机制，定时检测Pod的健康状态。定义一个HTTP GET请求，每隔一段时间检查一次Squid的运行状态。
* 日志管理：通过Kubernetes的PersistentVolume机制, 将Squid的日志挂载到宿主机，便于后续问题定位

**如何测试的**
8vCPU/12G/500G/1000Mbps
1000台agent并发升级, 流量157M*1000, 30分钟下载完毕
CPU 8%, Mem 58%, Disk 1%，峰值内存7G，瓶颈在带宽
157*8/(30*60) = 700Mbps, 相当于1Gpbs网卡

# 项目3: Matrix仿真平台
Matrix是一个基于Docker的单板仿真平台, 运用XML建模技术和Redis驱动仿真, 支持全产品形态四十多块单板的仿真. 该平台使得100多名开发和测试人员无需依赖物理设备即可进行软件测试.  
Matrix平台易于部署，可以轻松安装在每个开发人员的Linux机器上，显著提升了日常开发和测试效率, 并节省了物料成本。  

**背景**
在设备管理领域，传统的测试方法通常需要几台设备共享给上百个开发人员使用，每次测试排队时间很长, 测试一次耗时半小时以上，极大地限制了开发效率。
为了解决这一痛点，我们开发了Matrix仿真平台, 让每个开发可以在自己工作机器上快速测试代码, 节省排队和测试时间, 提高开发失效率
作为设备管理部门，我们的业务代码依赖于底层的驱动接口，因此在仿真环境中必须设计一种方法模拟这些驱动接口。 为了满足测试人员灵活构造各种故障的需求，驱动接口不能是硬编码的
整体架构

**Matrix平台的整体架构是什么样的？**
仿真平台部署在开发人员的Linux机器上，包含一个启动脚本, 一组模型文件, 一个仿真设备包
用户执行启动脚本，拉起一个管理容器, 这个管理容器中安装了Redis数据库，用于驱动接口仿真
管理容器读取模型文件, 获取需要拉起的仿真单板信息, 把单板属性写到Redis数据库, 再把相应的单板容器拉起来, 把仿真设备包解压到单板容器 

**XML建模技术具体如何应用的**
XML文件定义每个单板的属性, 比如单板类型, 在位状态,
管理容器启动后, 读取XML文件内容把单板数据以字符串形式写到Redis, 比如1号单板类型对应的key就是"/Chassis$#1/board#1/board_type"

**为什么选择Docker**
项目的主要目标是为开发和测试人员提供一个轻量级的仿真环境，而不是构建高可用、多集群的生产环境。
Docker使用简单，对开发和测试人员来说学习成本低

**为什么用Redis不用MySQL**
Redis是基于内存的数据库，性能非常高。本项目的数据规模大约为2万个key，QPS峰值可达1万左右，Redis可以满足需求
Redis的key-value模型非常适合模拟驱动接口的行为（如get/set操作），而MySQL作为关系型数据库需要先建表，灵活性较差。我们的驱动接口主要依赖于简单的键值操作，因此Redis更加灵活方便
Redis支持多种数据类型（如string、list、set、zset、hash），相比Memcached仅支持string类型，Redis功能更强大；而MongoDB是文档型数据库，更适合文档存储，不适合我们当前的应用场景
Redis支持发布订阅和键空间通知功能，我们在项目中利用Redis特性实现了单板插拔流程的仿真验证，无需引入其他中间件, 使系统简单易于维护

**Redis驱动仿真API的具体功能, 怎么实现动态故障注入***
在设备管理领域，测试人员需要一种简单且可靠的方法来动态修改驱动接口的行为，以便模拟各种故障场景进行测试
传统的做法是利用华为内部的一个C语言库进行函数动态插桩，但在我们设备管理领域，这种方法会出现替换失败的情况，不可靠
为了解决这些问题，我们设计了一种基于Redis的方案，通过Redis动态改变驱动接口行为，实现故障注入

具体来说，我们使用Hiredis（一个C语言开源库）开发了一组读写Redis key的API，用于模拟驱动接口的行为。
由于被测组件也是so库，集成Hiredis非常方便。例如，要将1号单板设置为“不在位”状态，只需修改Redis key "/board#1/present_status" 的值为 false 即可，无需修改驱动代码或重新编译。
相比旧的插桩方案，新方法不仅更加稳定可靠，还极大地简化了故障注入的操作流程，使得测试人员可以快速灵活地构造各种故障场景，显著提高了测试效率和可靠性。

## 通过Redis键空间通知机制, 模拟了单板插拔的仿真, 并通过自定义网桥模拟框式形态的仿真, 扩展了仿真平台的测试范围

**Redis键空间通知机制是如何帮助模拟单板插拔仿真的？**
为了在仿真环境测试单板插拔场景, 需要模拟相关驱动函数行为. 
在真实的设备管理业务中，当发生单板插拔事件时，驱动会通过回调通知设备管理进程。

在仿真环境中，我们需要模拟驱动行为, 如果直接用轮询方式检查Redis中的单板在位状态会导致不必要的CPU开销。 为了解决这一问题，我们利用了Redis的键空间通知机制，具体流程如下：
设备管理进程订阅1号单板是否在位的频道"Board1/present_status', 每当该键发生变化时，Redis都会发送通知。
模拟单板插入时，把表示单板在位的Redis key值置为on, 触发Redis键空间通知
Redis将通知消息发布给所有在线的消费者（即设备管理进程）。设备管理进程收到订阅消息后，触发业务回调，从而感知到单板插拔事件的发生。

利用Redis的键空间通知机制, 实现了单板插板场景仿真, 避免轮询方式带来CPU浪费， 同时没有引入额外中间件, 保持系统的简洁

**接口设计**
```
typedef void (*hiredisSubCallback) (hiredisSubContext *cxt);
int HiredisSubscribe(const char *key, hiredisSubCallback cb, hiredisSubContext *cxt);
```
* 订阅频道 subscribe  __keyspace@0__:/board#1/present_status"
* 创建线程，循环读取redis回复, 收到回复后调用回调


**自定义网桥是如何实现框式形态仿真的？**
在框式设备仿真中，单板间的通信需要使用特定的172.16网段, Docker默认的docker0网桥使用的是172.17
为了避免修改客户Docker配置或重启服务，我们创建了一个自定义网桥，指定其IP段为172.16，将所有单板容器连接到这个网桥上。 通过这种方式，容器间可以直接通过172.16 IP段通信，模拟了真实设备的网络环境。

Docker间通信原理是，每个容器通过一对虚拟以太网接口（veth对）与宿主机上的自定义网桥通信
veth pair由两个虚拟网络接口组成，一端位于宿主机的网络命名空间（network namespace），另一端位于容器的网络命名空间。
当容器启动时，Docker会自动创建veth pair，并将其一端连接到容器内部的eth0接口，另一端连接到宿主机上的自定义网桥。
这样，容器内的网络流量可以通过自定义网桥在不同容器之间传输，实现高效的局域网通信。

## 将Redis驱动接口从短连接优化为长连接, 接口平均读写时间从2ms缩短为0.1ms, 显著提升了测试效率

**将Redis驱动接口从短连接优化为长连接**
在项目初期，驱动接口通过Redis短连接实现，导致频繁建立和关闭连接。特别是在设备管理子卡初始化代码中，需要查询大量属性时，影响测试效率
为了解决这个问题，我们将短连接优化为长连接。 通过互斥锁, 让一个进程的所有线程共用一个长连接。 最终将驱动接口的平均一次读写时间从2ms缩短到0.1ms，显著提升了性能。
pthread_mutex

**如何测量和验证平均一次读写时间从2ms缩短至0.1ms的效果？**
在测试环境, 直接修改C代码, 用time_t计算时间差,


## 其他问题
**举一个问题定位案例**
当时遇到一个Redis key被意外删除, 导致驱动接口行为不符合预期的问题 

我们通过常规的查看日志, tcpdump抓包方法, 没有找到原因.
我当时看过REDIS的单机数据库实现, 想到既然Redis是基于内存的数据库，字符串键肯定在进程中的某个内存地址处，key丢失时这个地址的内容肯定被改写
所以我只需要GDB打个断点，看下调用栈不就搞定了
后来发现是REDIS配置项maxmemory过低, 只给了1M, 导致Redis认为空间不够就随机淘汰了一些key, 修改了这个配置项后问题得到了解决

**你负责哪些部分, 怎么和其他团队成员合作的**
这个工具是合作完成的
早期设计阶段, 有1个架构师, 1个SE, 我参与项目设计和方案讨论, 梳理业务做原型验证，和架构,SE讨论方案。
连我在内, 有两个开发，另一个开发来自杭州。 我负责驱动接口仿真和Redis，另一个同事负责Docker镜像, 仿真设备包构建

后期的性能优化也是合作的，连我在内有3名开发，在同一个部门的业务组。 我做方案设计, 梳理流程拓扑, 把开发任务分解到2名开发同事去做。

**你觉得还有什么优化**
* 优化内存。比如对字符串键做压缩存储（因为Redis字符串键的三种编码: long, embstr,raw）
* 优化请求速度，有些驱动函数会一次性读写多个key，用mset, mget命令代替get, set, 减少客户端和Redis的通信次数

**Redis挂了怎么办, 可靠性怎么考虑**
每个单板1个容器，作为redis client; 1个管理容器，安装redis数据库
如果Redis挂了，管理容器会检测到异常，并依次重启管理容器和各个单板容器。
由于这是仿真环境大部分驱动数据是易失性的，无需持久化或集群支持; 对于需要持久化数据, 通过写文件和Docker Volume方式存储


# 项目最大难点
* VA升级(无缝, 配置和服务同步)
* 网络问题定位, 针对客户防火墙问题优化

# 一些概念

## 什么是混合云架构
混合云架构是一种计算环境, 结合了本地部署和公有云资源, 这种架构可以减少成本, 提高系统的响应速度和服务可靠性 
以我们的服务网关项目为例, 客户的终端直接请求本地部署的服务网关, 不需要从云端获取数据，这样有很多好处，比如：
* 有效节省了客户带宽，并且降低了我们的公有云费用
* 服务在客户本地网络完成, 这样减少了网络延迟，提高了响应和客户体验。
* 还有安全性考虑, 比如像银行这样对安全要求高的客户, 终端不能直接访问外网，需要通过虚拟设备代理上网，防止内部网络直接暴露给外网, 同时实现集中的访问控制

## 支持XX多平台部署, 这些平台差异是什么, 遇到什么问题

## 你们的Microk8s集群是指什么? 为什么不用高可用集群? 高可用集群是什么样的?
我们的服务网关使用的集群是多个独立运行的Microk8s节点组成，每个节点都可以独立工作，比如一个节点负责处理上万个终端请求。

我们没有采用高可用的多主多从架构。 因为高可用方案需要配置多个主节点和工作节点, 配置和运维复杂, 导致客户资源浪费
我们的方案简单灵活，每个Microk8s节点独立工作, 配置简单, 易于扩展。

使用高可用集群的目的是避免单点故障，保证服务连续性。 采用多主节点架构。
每个主节点运行控制平面组件，包括apiserver, schedular, controller-manager, 当主节点或工作节点故障后，系统能自动检测并切换到备用节点

## XDR是什么意思
Extended Detection and Response 扩展检测和响应，是一种安全解决方案。 为企业客户提供全面的安全防护，从终端设备到云端服务的安全防护。
比如说我们的Service Gateway, 部署在企业网络，为本地终端提供一些核心的XDR服务，例如：
* ActiveUpdate 主动式更新, 更新最新的威胁情报库
* WebReputation 评估用户访问的网站安全性，防止恶意网站的威胁
* ForwardProxy 为客户终端上云提供了统一出口, 同时为终端提供了一个AirGap的解决方案. AirGap是值终端网络隔离,代理上云

## ActiveUpdate
我们的Active Update服务运行在本地的服务网关上, 定期从远程的服务器下载最新的病毒库，扫描引擎等文件。 
通过这种方式, 客户的终端产品可以直接访问本地的服务获取更新，不必直接连接到远程服务器. 这样好处是显著减少了客户的带宽压力 
比如有两万个终端需要下载同一份病毒库，使用本地的ActiveUpdate服务可以避免对客户网络造成冲击, 提高响应速度，减少成本。

## WebReputation
WebReputation是一个用于评估网页安全性的服务。 通过给访问的网站打分帮助用户避免安全威胁。
比如说，像Google, MS这种知名网站给一个高分加到白名单里; 恶意站点, 或者和白名单相似的可疑网站, 会给一个低分
这个数据是动态更新的，依赖公司的爬虫服务, 加上和第三方购买的一些数据


## JWT和Session
JWT无状态的token, 服务端不需保存; Session是有状态的, 需要存储Session识别用户
JWT是无状态的，它更易于扩展, 但是难以撤销, 通常通过HTTP头传输， Session依赖于cookie传输Session ID

Cookie和Session认证问题
```
1、用户向服务器发送用户名和密码。
2、服务器验证通过后，在当前对话（session）里面保存相关数据，比如用户角色、登录时间等等。
3、服务器向用户返回一个 session_id，写入用户的 Cookie。
4、用户随后的每一次请求，都会通过 Cookie，将 session_id 传回服务器。
5、服务器收到 session_id，找到前期保存的数据，由此得知用户的身份。
```

JWT 的原理是，服务器认证以后，生成一个JSON WEB TOKEN，发回给用户，用户与服务端通信的时候，都要发回这个TOKEN。服务器完全只靠这个对象认定用户身份。
为了防止用户篡改数据，服务器在生成这个对象的时候，会加上签名

Header.Payload.Signature

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

LAULAU (Local Active Update) service here is a containerized service which can be run on Service Gateway Virtual Appliance to play as a local AU server for TrendMicro on-premise products, like Apex One, DDI and DSM  etc.

Ability to regularly download components (pattern/engine) from specified global AU (per product type).
Allow point products to download components from it by AU/iAUSDK protocol on demand or by schedule by given service URL (per product type).
Support multiple point products (in different product type) concurrently.
Report all component version/last update etc info back to V1 SG app.
Report the count of component downloading per product back to V1 SG app.

## SG introduction:
A Service Gateway installed in the local network acts as a relay between Trend Micro Vision One and other products, such as on-premises Trend Micro or third-party products. 
This allows use of Trend Micro cloud services while reducing Internet traffic and sharing threat intelligence.

A Service Gateway consists of a cloud-based inventory list in the console app and a locally-based virtual appliance.

* Service Gateway Management
  Service Gateway Management provides status information on the connected Service Gateway virtual appliances, including appliance details, service statuses, and connected endpoints. 
  Service Gateway Management also allows you to manage the connected Service Gateway virtual appliances.

* Service Gateway virtual appliance
  The Service Gateway virtual appliance attached to the local network provides services like ActiveUpdate, Smart Protection Services, and Suspicious Object List synchronization to on-premises Trend Micro products. 
  The Service Gateway also supports log forwarding and integration of third-party applications with Trend Micro Vision One.


Service

Description

AWS

Azure (ADDA)

Owner Team

ActiveUpdate

Serves on-premises Trend Micro products as a local ActiveUpdate server to reduce outgoing internet traffic.

O

O

Service Gateway

Forward proxy

Allows agents on endpoints with no direct access to the internet to use the service gateway as a proxy to reach Trend Micro Vision One.

O

O

Service Gateway

Nessus Pro

Sends device information and vulnerability data from the Nessus Pro server to Trend Micro Vision One.

O

 

SASE

On-premises directory connection

Once enabled, the Service Gateway can help send data from on-premises directory servers to Trend Vision One. This service is required to set up "Active Directory (on-premises)", and "OpenLDAP" in Third-Party Integration

O

O

Active Directory

Rapid7 - Nexpose

When enabled, the Service Gateway can send device and vulnerability data from the Rapid7 server to Trend Vision One.

O

 

SASE

Tenable Security Center Connector

Once enabled, the Service Gateway sends device information and vulnerability data from the Tenable Security Center server to Trend Vision One.

O

 

SASE

Smart Protection Services

Leverages file reputation and web reputation technology to detect security risks. On-premises Trend Micro products can perform queries against the Service Gateway virtual appliance, which provides Smart Protection either through the local Smart Protection Server on the virtual appliance or as a reverse proxy.

O

O

SG SPS Team

Suspicious Object List synchronization

Supports the sharing of Suspicious Object lists between Trend Micro Vision One and on-premises Trend Micro products.

O

O

Threat Intel

Syslog Connector

Enables sharing data from Trend Micro Vision One with your local syslog server.

O

O

TPI

Third-party intelligence synchronization

Shares threat intelligence from Trend Micro Vision One with third-party applications or retrieves threat intelligence from third-party applications.

O

O

TPI

TippingPoint

Log forwarding

Supports forwarding logs to Trend Micro Vision One for correlation and analysis.

O

O

Service Gateway

TippingPoint policy management

Allows the Network Intrusion Prevention app to modify TippingPoint policy configurations to mitigate CVEs.

O

TippingPoint

Zero Trust Secure Access On-Premises Gateway

Zero Trust Secure Access Internet Access is a forward proxy service that protects end users from malicious activity on the internet. In addition to the Cloud Gateway, the on-premises gateway also provides a flexible option to deploy one or more local on-premises gateways in your organization's network as a hybrid protection solution.

O

ZTSA

Generic Caching Service

Saves previously requested internet content for on-premises Trend Micro products to speed up access to data and reduce demand on bandwidth of your business.

O

Service Gateway

Third-Party Log Collection Service

Supports sending logs collected from your local devices to Trend Vision One for correlation and analysis.

O

File Security Virtual Appliance

Scan valuableScan valuable files for malware files for malware

O