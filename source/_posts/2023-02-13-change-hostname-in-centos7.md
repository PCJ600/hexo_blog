---
layout: next
title: Centos7中修改hostname的方法
date: 2023-02-13 21:04:15
categories: Linux
tags: Linux
---

## hostname临时修改主机名
```bash
hostname XXX
```
## hostnamectl永久修改主机名
```bash
hostnamectl XXX
```
修改的内容实际是文件`/etc/hostname`

## 三种主机名的区别

<!-- more -->

```bash
man hostnamectl
This tool distinguishes three different hostnames: 
the high-level "pretty" hostname which might include all kinds of special characters (e.g. "Lennart's Laptop"), 
the static hostname which is used to initialize the kernel hostname at boot (e.g. "lennarts-laptop"), 
and the transient hostname which is a default received from network configuration. If a static hostname is set, and is valid (something other than localhost), then the transient hostname is not used.
```
## Centos7中，hostnamectl --static命令读取的实际是/etc/hostname文件的内容
`hostnamectl`源码参考：[https://github.com/systemd/systemd](https://github.com/systemd/systemd)
`hostnamectl --static`函数调用梳理如下：
```c
DEFINE_MAIN_FUNCTION(run);
hostnamectl_main func table
get_or_set_hostname
get_hostname_based_on_flag
get_one_name(bus, "StaticHostname", NULL);
sd_bus_get_property
sd_bus_call_method
sd_bus_call_methodv();
static const BusObjectImplementation manager_object = {
        "/org/freedesktop/hostname1",
        "org.freedesktop.hostname1",
        .vtables = BUS_VTABLES(hostname_vtable),
};
hostname_vtable
SD_BUS_PROPERTY("StaticHostname", "s", property_get_static_hostname, 0, SD_BUS_VTABLE_PROPERTY_EMITS_CHANGE)
property_get_static_hostname 
context_read_etc_hostname
read_etc_hostname
int read_etc_hostname(const char *path, char **ret) {
        _cleanup_fclose_ FILE *f = NULL;
        assert(ret);
        if (!path)
                path = "/etc/hostname";
        f = fopen(path, "re");
        if (!f)
                return -errno;
        return read_etc_hostname_stream(f, ret);
}
```
可以看出，最终调用`read_etc_hostname`读取`/etc/hostname`的内容
