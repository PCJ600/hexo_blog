---
layout: next
title: 如何定制RockyLinux ISO
date: 2024-11-05 20:28:02
categories: Linux
tags: Linux
---

# 实现目标
基于Rocky9官方ISO做定制，构建自己的ISO
* 可以添加非官方预装的RPM包，或者自定义RPM包
* 实现Kickstart自动化安装, 完成分区等操作
* ISO安装后，可以执行自定义脚本，比如安装你手动添加的RPM包

# Rocky9 官方ISO内容分析
挂载Rocky9 ISO，得到如下内容:
```
BaseOS/
EFI/
images/
isolinux/
LICENSE
media.repo
minimal/
```
ISO各个目录/文件的作用：
* BaseOS/：这个目录包含了Rocky Linux的基础操作系统环境。它提供了操作系统的核心组件和必要的软件包，用于构建和运行基本的系统
* EFI/：这个目录包含了用于UEFI（统一可扩展固件接口）启动的文件。这些文件使得系统能够在支持UEFI的硬件上启动
* images/：这个目录包含了用于云环境的Rocky Linux镜像。这些镜像可以被用于各种云服务提供商，以便于在云中部署Rocky Linux
* isolinux/：这个目录包含了启动Rocky Linux安装介质所需的引导装载器文件。这些文件负责在系统启动时加载Linux内核和初始化RAM磁盘
* minimal/：这个目录包含了用于最小化安装的Rocky Linux环境。它通常用于安装一个最小化的Rocky Linux系统，不包括完整的DVD镜像或者通过网络安装
* media.repo：这个文件是一个YUM仓库配置文件，它允许用户直接从安装介质（如DVD或USB驱动器）安装软件包。这个文件指定了安装介质中软件包的位置，使得系统能够从本地介质而不是网络仓库安装软件
* LICENSE：这个文件包含了Rocky Linux发行版的许可证信息。它说明了用户可以如何使用和分发Rocky Linux

# 定制ISO的流程简述
* 准备一台Rocky9.X编译机, 安装必要编译依赖, 下载Rocky官方的ISO, 挂载ISO。
* 把自定义的RPM包复制到Packages/目录下, 调用createrepo更新RPM信息。
* 编写ks.cfg,实现kickstart定制化的自动安装
* 调用genisoimage生成VA的ISO, 调用implantisomd5校验ISO的md5

# 定制ISO的具体方法

## 0.准备一台RockyLinux9.X编译机，安装必要的编译依赖
```
dnf install -y epel-release genisoimage isomd5sum anaconda \
createrepo mkisofs rsync yum-utils squashfs-tools rpm-build \
glibc-devel krb5-devel autoconf automake pykickstart
```

## 1.下载Rocky9.4 minimal官方ISO, 挂载ISO
```
wget https://download.rockylinux.org/pub/rocky/9/isos/x86_64/Rocky-9.4-x86_64-minimal.iso
mkdir tmp ISO
mount -o loop Rocky-9.4-x86_64-minimal.iso tmp/
cp -a tmp/* ISO/
```

## 2. 添加非预装的RPM包到ISO
添加RPM包的原理是
* 启动一台新安装的Rocky 9.4的虚拟机或容器, 执行`rpm -qa > old_rpm_list`记录当前系统的所有RPM包
* 执行yum install安装RPM包, 安装成功后, 再次执行`rpm -qa > new_rpm_list`
* 通过diff命令比较new_rpm_list和old_rpm_list的差异, 得到这个RPM包依赖的所有RPM包, 把所有RPM包下载到本地
* 最后把RPM包都传到ISO的Packages目录, 调用createrepo更新RPM信息	

这里以安装Squid为例, 给出详细步骤:
新安装一台Rocky9.4的VM，或者启动一个新的Rocky9.4容器，执行如下操作:
```
rpm -qa > old_rpm_list
yum install -y squid 
rpm -qa > new_rpm_list

# old_rpm_list和new_rpm_list的差异就是需要手动下载并上传到ISO的RPM包
diff old_rpm_list new_rpm_list
> perl-English-1.11-481.el9.noarch
> perl-Math-Complex-1.59-481.el9.noarch
> perl-Math-BigInt-1.9998.18-460.el9.noarch
> perl-DBI-1.643-9.el9.x86_64
> libecap-1.0.1-10.el9.x86_64
> perl-Digest-SHA-6.02-461.el9.x86_64
> httpd-filesystem-2.4.57-11.el9_4.1.noarch
> squid-5.5-13.el9_4.x86_64

# 逐一下载上面的RPM包
yum reinstall -y [RPM包] --downloadonly --downloaddir=./

# 把所有RPM包传到ISO/minimal/Packages目录下

# 调用createrepo更新RPM信息, 删掉原来的xml
cd ISO/minimal/
createrepo -g repodata/*.xml ./

执行成功后，查看repodata目录, 可以手动把旧的文件删掉
# ll -rt
-rw-r--r-- 1 root root  223063 May  6  2024 b293fe7dc4f43936b010f1e39498d1c1914ec9b64b239ca295a99024469ad291-other.xml.gz
-rw-r--r-- 1 root root  639692 May  6  2024 910d1caf3c2f6f2be8ec8d1690ade00b3ca742ff265c58bfe8e09564907c7b1f-primary.xml.gz
-rw-r--r-- 1 root root  432959 May  6  2024 6727bfc392a5f0d542bfa33e2e7c66c0acfac68ca643a815ff142eceb83c14eb-filelists.xml.gz
-rw-r--r-- 1 root root  277890 May  6  2024 4844662615d6cdbd59e187e2ff1f4d3aa7fbc153dd28b6024c7eef14e9e14c94-other.sqlite.bz2
-rw-r--r-- 1 root root  546603 May  6  2024 33df6249457b6e40f26bcee5eee9fbecbdb8026b50c9ce3a354001ea04fcaa9d-filelists.sqlite.bz2
-rw-r--r-- 1 root root 1338121 May  6  2024 bd201f63f99e67d65f859f38ab472022f055238d74c78c6dd407ef57c4f0f90d-primary.sqlite.bz2

-rw-r--r-- 1 root root  288689 Nov  5 22:04 7ec709b6d42da53e7fb35b426e69414a518151eb4814b1fe1703b7b18d519e33-x86_64.xml
-rw-r--r-- 1 root root  644366 Nov  5 22:04 aa896d64b33075d63a1a150b45f78069845b6ba23e1d9b6914028c89d3444343-primary.xml.gz
-rw-r--r-- 1 root root  446349 Nov  5 22:04 96bac81c16728c9c33f8a7da9388ceaeb8fa19a0d173fd3220ca34f5e91e5966-filelists.xml.gz
-rw-r--r-- 1 root root  225673 Nov  5 22:04 4b548eb44fe93e0561e5c451d5d26ae93033ba268276eb953c2a88f3d898d554-other.xml.gz
-rw-r--r-- 1 root root   71732 Nov  5 22:04 d250f7f881bb991be3648c021fb305dd6085b902321b26f52033500ebff7cae1-x86_64.xml.gz
-rw-r--r-- 1 root root  279932 Nov  5 22:04 35a6d5480475ae34480bd0b8eeeeb4954135b980d7585f645f92adc563af724b-other.sqlite.bz2
-rw-r--r-- 1 root root  554441 Nov  5 22:04 131a105582b0b396b43bfe2085074488ec256b9901e115f1699d9ef40f0cbfce-filelists.sqlite.bz2
-rw-r--r-- 1 root root 1352325 Nov  5 22:04 fb6353f4de5dacf439dd366b2caf0d7f8fff4da3dca132dbbd86b820bbed0e28-primary.sqlite.bz2
```

## 3. 添加ks.cfg，实现Kickstart自动化安装ISO
* 自定义分区, gpt格式, biosboot+/boot+volgroup(/ /data /back)
* 设置root账号，密码password@123
* 设置hostname为localhost.localdomain
* 网卡名称固定为eth0, static IP 192.168.192.3, 掩码255.255.255.252, 网关192.168.192.2, DNS8.8.8.8
* 禁用firewall
* 关闭selinux
* 安装自定义的RPM包
* 执行自定义指令

装一台RockyLinux9.4，在图形安装界面手动完成以上配置，安装完成后把/root/anaconda-ks.cfg拷出来，文件名改成ks.cfg
ks.cfg内容如下：
```
```

检查ks.cfg语法正确, 再把ks.cfg拷贝到ISO目录
```
cp ks.cfg isobuild/isolinux/ks.cfg
```


## 修改isolinux.cfg, grub.cfg
isolinux.cfg 
把默认启动菜单去掉, 添加kickstart方式启动菜单，inst.ks指定ks.cfg
```

```

grub.cfg
修改menuentry, inst.ks指定ks.cfg
```
```

## 构建ISO
调用genisoimage生成ISO, 调用implantisomd5校验ISO的md5
```
genisoimage -input-charset utf-8 -U -r -v -T -J -joliet-long -V "VA" -volset "VA" -A "VA" \
-b isolinux/isolinux.bin -c isolinux/boot.cat -no-emul-boot -boot-load-size 4 -boot-info-table \
-eltorito-alt-boot -e images/efiboot.img -no-emul-boot -o custom.iso ISO/

implantisomd5 custom.iso
```

## 测试, 安装ISO
VMware WorkStation 安装ISO, 测试ISO自动安装是否成功

## 参考资料
