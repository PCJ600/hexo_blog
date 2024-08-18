---
layout: next
title: Linux内核模块编译方法
date: 2020-08-15 18:06:06
categories: Linux
tags: Linux
---

### 运行环境

`Linux debian 4.19.0-10-amd64`

### 编译内核模块

#### 0. 准备编译所需的内核头文件

系统默认内核头文件路径在/lib/modules/\`uname -r\`，先确认该路径是否存在：

```shell
ls /lib/modules/`uname -r`/build
```

如路径不存在，需要先安装内核头文件，方法如下：

<!-- more -->

* 获取内核版本，使用`uname -r`查看，这里为`4.19.0-10-amd64`

* `apt search 4.19.0-10-amd64`查找安装包名称，这里为`linux-headers-4.19.0-10-amd64`

* 安装内核头文件，执行`apt-get install linux-headers-4.19.0-10-amd64`，安装路径为`/usr/src/linux-headers-4.19.0-10-amd64`

#### 1. 编写`hello.c`

```C
#include <linux/module.h>           // 编译内核模块必须加载的头文件module.h
#include <linux/kernel.h>
#include <linux/init.h>

MODULE_LICENSE("GPL");

static int __init hello_init(void)
{
    printk(KERN_INFO "hello_init\n");
    return 0;
}

static void __exit hello_exit(void)
{
    printk(KERN_INFO "hello_exit\n");
}

module_init(hello_init);    // 模块加载
module_exit(hello_exit);    // 模块卸载
```

模块加载时打印`hello_init`, 模块卸载时打印`hello_exit`。

#### 2.编写Makefile

```Makefile
ifneq ($(KERNELRELEASE),)
obj-m += hello.o    						# -m编译内核模块
else
PWD=$(shell pwd)
KDIR=/usr/src/linux-headers-4.19.0-10-amd64 # 指定内核头文件路径
all:
    $(MAKE) -C $(KDIR) M=$(PWD) modules     # -C指定内核头文件路径, M指定源码路径
clean:
    $(MAKE) -C $(KDIR) M=$(PWD) clean
endif
```

`PWD`指定源码路径，即`hello.c`的路径。

`KDIR`指定内核源码路径。

`KERNELRELEASE`是在内核源码的顶层Makefile里定义的变量，用法可参考[这篇文章](https://blog.csdn.net/cjluxuwei/article/details/37878021)

#### 3.加载模块

加载ko：`insmod hello.ko`

卸载ko：`rmmod hello.ko`

查看ko是否加载：`lsmod | grep hello.ko`

打印信息如下：

```
# dmesg -c
[  821.764791] hello_init
[  897.219392] hello_exit
```




