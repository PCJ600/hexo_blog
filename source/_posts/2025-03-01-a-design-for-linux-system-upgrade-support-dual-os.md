---
layout: next
title: 一种Linux系统在线迁移方案的设计与实现
date: 2025-03-01 16:24:19
categories: Linux
tags: Project
---

# 背景
客户有一些on-premises的virtual appliance(VA)需要做固件升级。 这些VA是Centos7的OS，而Centos7已经停止维护，我们计划把客户的VA升级到新的OS(RockyLinux9.4)。
当时客户VA升级固件的方式是增量升级(差分升级), 只有一个系统分区。 增量升级存在一些难以解决的问题:
* 无法实现系统软件的升级。 由于Rocky9.4和Centos7的系统软件包差异太大，卸载这些RPM包会直接导致系统不可用
* 升级出现故障后，无法做到版本回滚，只能让客户重装。 难以回滚的原因是升级前后用的是同一个系统分区，数据不隔离。

因此，需要设计一种全量升级的方案，把客户的VA升级到Rocky9.4这个新的OS <br/>
<!-- more -->
全量升级是指, 做一个全量升级包, 客户的VA通过网络下载升级包，解开升级包后执行一些脚本, 切换到新的OS启动, 完成固件升级。

# 难点
* 需实现一键升级, 即客户除了在UI上点升级，无需做其他任何的手动操作
	* 无需让客户手动调整磁盘容量, 或者添加新的虚拟磁盘
	* 切换到新系统后, 可以自动同步原先的系统配置，例如: IP, DNS, 登录口令, 而不是让客户手动配置
* 升级后需支持双系统启动，但是客户的VA只有一个系统分区。 这涉及到重新建立分区的操作。

# 方案
基于initrd做一个升级小系统, 先进入升级小系统重建分区，再安装新系统, 流程:
* 做一个升级包，内容包括: 一个包含新系统所有文件的压缩包、 分区表文件、 升级小系统(vmlinuz,定制的initramfs)
* 客户的VA下载并解压这个升级包到硬盘
* 备份当前系统配置到硬盘(切换到新系统后, 需要根据这个备份信息恢复系统配置)
* 调整GRUB的默认启动项，从升级小系统启动(根据Rocky9官方ISO的紧急修复模式修改得到的系统), 重启后进入升级小系统的启动流程:
	* 挂载根分区, 把升级包和系统配置复制到内存(临时文件系统)
	* 对整个硬盘重建分区，以支持双系统分区
	* 把新系统的所有文件解压到根分区
	* 恢复系统配置和登录口令
	* chroot到根分区, 重装GRUB后重启，完成从升级小系统到新系统的切换

# 实现

## 制作升级包
升级包就是一个tar包(migration.tar.gz)，客户的VA通过网络下载、解压升级包, 执行其中的脚本自动完成升级。 升级包的内容如下：
```bash
# tar -tvf migration.tar.gz
va.sgdisk                   # 使用sgdisk导出的新系统的分区表文件，支持双系统启动
va.tar.xz                   # 一个压缩包, 把RockyLinux9.4虚拟机导出到OVA，再把OVA文件系统中根分区下的所有文件备份成压缩包
initramfs-convert.img       # 基于Rocky9官方ISO的initramfs定制, 用于启动升级小系统
vmlinuz-convert             # 一个压缩内核, 直接从Rocky9官方ISO的vmlinuz拷贝，用于启动升级小系统
upgrade_to_tiny_system.sh   # 一个Shell脚本，在客户VA上解压升级包后，自动执行这个脚本，切换到升级小系统启动 
```
以下说明这些升级包中的文件如何制作

### 制作va.sgdisk
首先，基于Rocky9.4的ISO安装一台VMware虚拟机。 注意创建分区时我们只分10个G空间，目的是让导出的OVA文件尽可能小, 可以在客户把新系统安装完毕后，把剩余的磁盘扩容。
新系统采用BIOS/GPT分区格式, 原因是:
* 使用BIOS，而不是UEFI，主要是为了兼容老用户
* 相比于MBR, GPT能支持更大容量的磁盘，可靠性和性能也更好

分区方案是: 1个biosboot分区 + 1个boot分区 + 1个LVM(系统分区)，如下:
```
/dev/sda1 BIOS boot 2 MiB
/dev/sda2 /boot     1 GiB
/dev/sda3 LVM       9 GiB
```
说明: 
* BIOS/GPT启动，要求必须要有一个BIOS boot分区
* 给/boot单独分区不是必要的，也可以不分; 一个/boot分区可以启动多个内核，没有必要为了支持双系统分两个boot区。
* 系统分区设计为LVM, 而不是标准分区, 这样可以灵活扩容
* 只分10G的容量, 剩余的硬盘空间等客户升级完成后再做扩容, 这样可以把升级包尽量做小(大小控制在1G左右)

对于LVM分区, 先创建一个名为VA的逻辑卷组，基于VA逻辑卷组再分出三个逻辑卷(VA-root, VA-back, VA-data)
```
NAME		SIZE      MOUNTPOINTS
sda3		
|- VA-root  8G        /
|- VA-back  512M      /back
|- VA-data  512M      /data
```
说明:
* 分三个逻辑卷的是为了实现双系统双启动项。如果升级新系统遇到故障, 可以迅速回滚到旧的操作系统。
* VA-root为第1个系统分区, VA-back是第2个系统的分区, 这两个系统共用一个/boot分区
* 设计VA-data分区，存储两个系统都会用到的公共数据

虚拟机安装完成后，使用gisk查看分区内容
```bash
yum install -y gdisk
gdisk -l /dev/sda
...
Disk /dev/sda: 20971520 sectors, 10.0 GiB
...
Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048            6143   2.0 MiB     EF02  BIOS boot partition
   2            6144         2103295   1024.0 MiB  8300  Linux filesystem
   3         2103296        20781055   8.9 GiB     8E00  Linux LVM
```
注：如果出现biosboot分区类型不识别的问题，可以用gdisk手动修改分区类型为biosboot，方法如下:
```bash
gdisk

Type device filename, or press <Enter> to exit: /dev/sda

Command (? for help): t
Partition number (1-3): 1

Hex code or GUID (L to show codes): EF02
Changed type of partition to 'BIOS boot partition'

Command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING PARTITIONS!!

Do you want to proceed? (Y/N): Y
OK; writing new GUID partition table (GPT) to /dev/sda.
```
再重新安装GRUB，reboot确认下系统启动没有问题
```bash
grub2-install /dev/sda
reboot
```
至此, 确认分区无误后，使用gdisk导出分区表文件`va.sgdisk`，这个分区表文件在后续升级的时候会用到
```bash
yum install -y gdisk
sgdisk --backup=./va.sgdisk /dev/sda
```



### 制作va.tar.xz
1、导出RockyLinux9.4虚拟机的OVA文件，解压OVA得到一个vmdk格式的硬盘文件
```bash
tar xf va.ova
ls va-disk1.vmdk
```
2、使用guestmount命令挂载出上面的vmdk文件，再对系统分区和boot分区的内容打包，得到va.tar.xz 。注意打包时需要保存xattr和uid
```bash
yum install -y libguestfs libvirt
systemctl start libvirt
export LIBGUESTFS_BACKEND=direct
	
mkdir -p  /mnt/vmdk
guestmount -a va-disk1.vmdk -i --ro /mnt/vmdk
XZ_OPT='-T0 -6' tar Jcpf va.tar.xz --numeric-owner --xattrs --xattrs-include=* --exclude='./back/*' --exclude='./data/*'  --exclude='./lost+found/*' --exclude='./tmp/*' -C /mnt/vmdk
```

### 制作initramfs, vmlinuz
制作initramfs, vmlinuz，用于启动一个升级小系统。 借助升级小系统实现重建分区表的同时，不丢失升级包和系统配置文件。
* vmlinuz —— 直接用Rocky官方ISO的vmlinuz即可，不需要对这个压缩内核做任何修改。 这里重命名为vmlinuz-convert
* initramfs —— 需要基于Rocky官方ISO的initramfs.img做定制, 再重命名为initramfs-convert.img

升级小系统启动流程设计如下: 
* 把va.tar.xz和备份的系统配置复制到内存(initramfs)
* 重建硬盘分区，以支持双系统双启动项
* 把va.tar.xz解压到硬盘的系统分区，完成新系统文件的安装
* 最后做一些必要的配置，确保从升级小系统切换到新系统启动

我们需要编写一个脚本(tiny_os_start.sh)，把这个脚本做到initramfs里。这样升级小系统启动时，可以通过这个脚本执行以上的流程。
#### 编写tiny_os_start.sh

##### 1. 把和系统配置文件复制到内存(initramfs)
首先把根分区挂载出来, 把升级包和旧的系统配置文件拷贝到内存。 考虑到重建分区后磁盘格式化，所有磁盘文件都会丢失，所以先把这些文件拷贝到临时内存里。
```bash
function load_upd_files_to_memory() {
  lvm vgscan --mknodes
  lvm vgchange -ay

  local rootdev=$(find /dev/mapper/ -name "*root")
  mkdir -p /sysimage
  mount "$rootdev" /sysimage

  # load upd files to memory
  cp -a /sysimage/etc/upd/* /etc/upd/ # 把升级包和旧的系统配置拷贝到内存
  umount -l -f /sysimage
}
```

##### 2. 重建磁盘分区
使用分区表文件(va.sgdisk)重建分区，再创建逻辑卷组和逻辑卷，格式化分区
```bash
function init_disk_partition() {
    # 重建分区
    local vgname=$(lvm vgs  | grep -v '#' | awk '{print $1}')
    local dskname=$(lvm pvs | grep '/dev/' | awk '{print $1}')
    lvm vgchange -an $vgname
    lvm pvremove --force --force -y $dskname
    sgdisk -Z /dev/sda
    sgdisk -l /etc/upd/va.sgdisk /dev/sda
    echo 1 > /sys/block/sda/device/rescan

    # 创建逻辑卷组VA和三个逻辑卷(data,back,root)
    lvm vgcreate VA  /dev/sda3
    lvm lvcreate -y -n data -L 512M VA
    lvm lvcreate -y -n back -L 512M VA
    lvm lvcreate -y -n root -l 100%FREE VA

    # 格式化分区
    mkfs.ext4 -F /dev/sda2
    mkfs.ext4 -F /dev/mapper/VA-root
    mkfs.ext4 -F /dev/mapper/VA-back
    mkfs.ext4 -F /dev/mapper/VA-data
}
```
注: initramfs.img中并没有sgdisk程序, 你需要拷贝sgdisk及其依赖到initramfs.img对应的bin,lib目录下

##### 3. 把va.tar.xz解压到硬盘的系统分区，完成新系统文件的安装
va.tar.xz包括新系统根分区下所有的文件以及boot分区的文件。 依次挂载系统分区(VA-root分区)和boot分区, 再va.tar.xz解压到磁盘
```bash
function extract_upd() {
    mount /dev/mapper/VA-root /sysimage
    mount /dev/sda2 /sysimage/boot
    tar xf /etc/upd/va.tar.xz --numeric-owner --xattrs --xattrs-include=* -C /sysimage"
    rm -f /etc/upd/va.tar.xz
}
```

##### 4. 做一些必要的配置，确保从升级小系统切换到新系统启动
单纯的解压文件到磁盘，是没办法切换到新的OS正常启动的。 必须要做一些配置操作，包括：
* 调整分区UUID。 boot分区的UUID要和系统分区的/etc/fstab中的保持一致。
* 同步内存中的LVM配置文件到系统分区。 因为配置LVM是在小系统里做的，小系统只是个临时的内存，真正的系统启动需要读取系统分区(VA-root)中的LVM配置
* 根据之前备份的系统配置，恢复新系统的配置。 例如: 网卡IP, 登录口令等。 
* chroot到系统分区, 重新生成initramfs, 重新安装GRUB，再reboot，完成从小系统到新系统的切换。

以上配置操作的实现可以参考:
```bash
function install_new_va() {
    # 调整boot分区的UUID, 重新挂载boot分区
    umount /sysimage/boot
    e2fsck -y -f /dev/sda2
    tune2fs -U $(grep UUID /sysimage/etc/fstab | awk '{print $1}' | sed 's/UUID=//') /dev/sda2
    mount /dev/sda2 /sysimage/boot

    # 同步LVM配置文件到系统分区
    cp -a /etc/lvm/backup/VA /sysimage/etc/lvm/backup/
    rm -rf /sysimage/etc/lvm/archive/
    cp -a /etc/lvm/archive/ /sysimage/etc/lvm/
   
    # 把旧系统的配置文件同步到系统分区
    ln -s /sysimage/etc/upd/ /etc/upd
   
    # 使用chroot重新生成initramfs，安装GRUB
    mount -t proc  /proc /sysimage/proc/
    mount -t sysfs /sys  /sysimage/sys/
    mount --rbind  /dev  /sysimage/dev/
    mount --rbind  /run  /sysimage/run/
	
    export KENREL_VER=$(ls /sysimage/boot/ | grep vmlinuz-5 | sed 's/vmlinuz-//')
    chroot /sysimage /usr/sbin/grub2-install /dev/sda
    chroot /sysimage /usr/bin/dracut -f /boot/initramfs-${KENREL_VER}.img ${KENREL_VER}
	
    # 重启，切换到新系统
    reboot
}
```
注: initramfs.img中并没有e2fsck、tune2fs程序，你需要手动拷贝这些程序及其依赖到initramfs.img对应的bin,lib目录下

完整的tiny_os_start.sh如下:
```bash
#!/bin/bash
function main() {
	load_upd_files_to_memory
	init_disk_partition
	extract_upd
	install_new_va
}
```

#### 自定义一个Systemd服务，用于小系统启动时自动执行tiny_os_start.sh
光有了脚本还不行，我们需要在小系统启动的某个时点执行这个脚本，可以通过自定义一个systemd的service实现。
经过实测，可以直接修改initramfs中已有的dracut-emergency.service，把ExecStart入口设置为tiny_os_start.sh即可，完整的dracut-emergency.service参考:
```
[Unit]
Description=Dracut Emergency Shell
DefaultDependencies=no
After=systemd-vconsole-setup.service
Wants=systemd-vconsole-setup.service
Conflicts=shutdown.target emergency.target

[Service]
WorkingDirectory=/
ExecStart=/sbin/tiny_os_start.sh # 指定你的脚本路径
Type=oneshot
StandardInput=tty-force
StandardOutput=inherit
StandardError=inherit
KillMode=process
IgnoreSIGPIPE=no

# Bash ignores SIGTERM, so we send SIGHUP instead, to ensure that bash
# terminates cleanly.
KillSignal=SIGHUP
```
接下来，需要把我们定义的脚本和service配置文件打到initramfs.img中

#### 定制initramfs.img
定制initramfs.img的流程如下:
* 解压Rocky官方ISO中的initramfs.img
* 把dracut-emergency.service, tiny_os_start.sh及其依赖(sgdisk、e2fsck、tune2fs等)拷贝到initramfs.img
* 最后把initramfs.img压缩回去

可以写个脚本(gen_initramfs.sh)实现以上流程
##### 0. 在你的Linux编译机上准备如下文件
```
# tree
gen_initramfs.sh 
resources/
	|- va.sgdisk
	|- initramfs-iso.img
	|- tiny_os_start.sh
	|- dracut-emergency.service
output/
```

##### 1. 解压initramfs.img
```bash
ROOT_DIR="$(dirname $(readlink -f -- "${BASH_SOURCE[0]:-$0}"))"
RESOURCES_DIR=${ROOT_DIR}/resources
OUTPUT_DIR=${ROOT_DIR}/output/img
DESTDIR=${OUTPUT_DIR}/rootfs_cpio

function extract_initramfs() {
	mkdir -p ${OUTPUT_DIR}/rootfs_cpio && cd ${OUTPUT_DIR}/rootfs_cpio
	export XZ_OPT='-T0 -6'
	xz -dc ${RESOURCES_DIR}/initramfs-iso.img | cpio -id
}
```

##### 2. 把dracut-emergency.service, tiny_os_start.sh及其依赖(sgdisk、e2fsck、tune2fs等)拷贝到initramfs.img
```bash
function modify_initramfs() {
	cp -a ${RESOURCES_DIR}/tiny_os_start.sh ${DESTDIR}/sbin/
	chmod 777 ${DESTDIR}/sbin/tiny_os_start.sh
	
	# 拷贝tune2fs、mkfs.ext4、sgdisk程序及其依赖的所有库
	cd ${DESTDIR}
	copy_exec_and_deps /usr/sbin/tune2fs /sbin
	copy_exec_and_deps /usr/sbin/mkfs.ext4 /sbin
	copy_exec_and_deps /usr/sbin/sgdisk /sbin

    cp /etc/mke2fs.conf ./etc/mke2fs.conf
    cp -a ${RESOURCES_DIR}/dracut-emergency.service ${DESTDIR}/usr/lib/systemd/system/dracut-emergency.service
}
```
注: copy_exec_and_deps函数用于拷贝一个可执行文件以及依赖的库，代码参考:
```bash
# $1 = file type (for logging)
# $2 = file to copy to initramfs
# $3 (optional) Name for the file on the initramfs
# Location of the image dir is assumed to be $DESTDIR
# If the target exists, we leave it and return 1.
# On any other error, we return >1.
copy_file() {
  local type src target link_target

  type="${1}"
  src="${2}"
  target="${3:-$2}"

  [ -f "${src}" ] || return 2

  if [ -d "${DESTDIR}/${target}" ]; then
    target="${target}/${src##*/}"
  fi

  # Canonicalise usr-merged target directories
  case "${target}" in
    /bin/* | /lib* | /sbin/*) target="/usr${target}" ;;
  esac

  # check if already copied
  [ -e "${DESTDIR}/${target}" ] && return 1

  mkdir -p "${DESTDIR}/${target%/*}"

  if [ -h "${src}" ]; then
    # We don't need to replicate a chain of links completely;
    # just link directly to the ultimate target
    link_target="$(readlink -f "${src}")" || return $(($? + 1))

    # Update source for the copy
    src="${link_target}"

    # Canonicalise usr-merged target directories
    case "${link_target}" in
      /bin/* | /lib* | /sbin/*) link_target="/usr${link_target}" ;;
    esac

    if [ "${link_target}" != "${target}" ]; then
      [ "${verbose?}" = "y" ] && echo "Adding ${type}-link ${target}"

      # Create a relative link so it always points
      # to the right place
      ln -rs "${DESTDIR}/${link_target}" "${DESTDIR}/${target}"
    fi
    # Copy the link target if it doesn't already exist
    target="${link_target}"
    [ -e "${DESTDIR}/${target}" ] && return 0
    mkdir -p "${DESTDIR}/${target%/*}"
  fi

  [ "${verbose}" = "y" ] && echo "Adding ${type} ${src}"

  cp -pP "${src}" "${DESTDIR}/${target}" || return $(($? + 1))
}

copy_libgcc() {
  local libdir library

  libdir="$1"
  for library in "${libdir}"/libgcc_s.so.[1-9]; do
    copy_exec "${library}" || return
  done
}

# $1 = executable/shared library to copy to initramfs, with dependencies
# $2 (optional) Name for the file on the initramfs
# Location of the image dir is assumed to be $DESTDIR
# We never overwrite the target if it exists.
copy_exec_and_deps() {
  local src target x nonoptlib ret

  src="${1}"
  target="${2:-$1}"

  copy_file binary "${src}" "${target}" || return $(($? - 1))

  # Copy the dependant libraries
  for x in $(env --unset=LD_PRELOAD ldd "${src}" 2>/dev/null | sed -e '
                /\//!d;
                /linux-gate/d;
                /=>/ {s/.*=>[[:blank:]]*\([^[:blank:]]*\).*/\1/};
                s/[[:blank:]]*\([^[:blank:]]*\) (.*)/\1/' 2>/dev/null); do

    # Try to use non-optimised libraries where possible.
    # We assume that all HWCAP libraries will be in tls,
    # sse2, vfp or neon.
    nonoptlib=$(echo "${x}" | sed -e 's#/lib/\([^/]*/\)\?\(tls\|i686\|sse2\|neon\|vfp\).*/\(lib.*\)#/lib/\1\3#')
    nonoptlib=$(echo "${nonoptlib}" | sed -e 's#-linux-gnu/\(tls\|i686\|sse2\|neon\|vfp\).*/\(lib.*\)#-linux-gnu/\2#')

    if [ -e "${nonoptlib}" ]; then
      x="${nonoptlib}"
    fi

    # Handle common dlopen() dependency (Debian bug #950254)
    case "${x}" in
      */libpthread.so.*)
        copy_libgcc "${x%/*}" || return
        ;;
    esac

    copy_file binary "${x}" || {
      ret=$?
      [ ${ret} = 1 ] || return $((ret - 1))
    }
  done
}
```

##### 3. 重新打包initramfs
```bash
function repack_initramfs() {
	cd ${DESTDIR}
	find . | cpio -c -o | xz -9 --format=lzma >/tmp/initramfs-convert.img
	cp -a /tmp/initramfs-convert.img ${OUTPUT_DIR}/
}
```

完整的gen_initramfs.sh参考:
```bash
function gen_initramfs() {
	extract_initramfs
	modify_initramfs
	repack_initramfs
}
```
至此, 升级小系统制作完毕。 我们还需要编写一个脚本(upgrade_to_tiny_system.sh), 客户VA解压升级包后执行这个脚本, 从而切换到升级小系统启动。

### 编写upgrade_to_tiny_system.sh，实现从老系统进入升级小系统 
编写upgrade_to_tiny_system.sh
```bash
RUN_DIR="$(dirname $(readlink -f -- "${BASH_SOURCE[0]:-$0}"))"

# 把升级小系统的initramfs,vmlinuz拷贝/boot分区
mv ${RUN_DIR}/vmlinuz-convert /boot/
mv ${RUN_DIR}/initramfs-convert.img /boot/
 
mkdir -p /etc/upd/
mv ${RUN_DIR}/va.tar.xz /etc/upd/
mv ${RUN_DIR}/va.sgdisk /etc/upd/

# 使用grubby，把默认启动项设置为升级小系统 
grubby --add-kernel=/boot/vmlinuz-convert --title="VA Migration" --initrd=/boot/initramfs-convert.img --args="initrd=initramfs-convert.img rd.retry=20 rescue"
grubby --set-default /boot/vmlinuz-convert
reboot
```

至此, 所有的编码已完成。 最后把升级包打出来(migration.tar.gz)即可。
```
# tar -tvf migration.tar.gz
va.sgdisk
va.tar.xz
initramfs-convert.img
vmlinuz-convert
upgrade_to_tiny_system.sh
```

升级流程:
客户的VA从网络下载升级包 -> 解压升级包, 执行upgrade_to_tiny_system.sh -> 进入升级小系统 -> 切换到新系统(RockyLinux9.4)

## 调试方法
难点主要是升级小系统的调试。 可以进入GRUB rescue模式手动加载内核, 手动挂载根分区，再逐步调试。

例: GRUB rescue模式手动加载内核
```
set root=(hd0,gpt2)
linux (hd0,gpt2)/vmlinuz-convert root=/dev/mapper/VA-root ro rd.lvm.lv=VA/root
initrd (hd0,gpt2)/initramfs-convert.img
```
如果需要手动挂载根分区, 可以故意指定一个错误的linux指令
```
linux (hd0,gpt2)/vmlinuz-convert root=/dev/mapper/undefined 
```
其他的调试手段可以自行Google

## 典型问题
问题: 切换到新系统后, 输入正确的用户名和密码，无法登录, console打印localhost login，没有任何错误提示。 而输入错误的密码会提示你认证错误。 进不了后台导致定位有困难。

解决方法: 通过手动调试升级小系统，手动挂载磁盘分区查看相关日志，发现是SELinux的问题。 设置新系统的SELinux为disabled后，问题得到解决。