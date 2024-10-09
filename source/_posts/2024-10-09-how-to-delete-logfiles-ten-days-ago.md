---
layout: next
title: Linux中定时删除10天前的日志文件
date: 2024-10-09 19:47:30
categories: Linux
tags: Linux
---

例如：删除/data/log/目录下所有10天前的.log文件
```
find /data/log/ -type f -name "*.log" -mtime +10 -exec rm -f {} \;
```

只查看要删除的文件有哪些，不真正删除文件
```
logfiles=$(find /data/log/ -type f -name "*.log" -mtime +10)
echo $logfiles
```
<!-- more -->

使用crontab添加一个定时任务，每天0点执行一次删除任务

先写个脚本`delete_old_logfile.sh`删除10天前日志
```bash
#!/bin/bash
export PATH=/usr/sbin/:$PATH

logfiles=$(find /data/log/ -type f -name "*.log" -mtime +10)
if [ -n "${logfiles}" ]; then
    find /data/log/ -type f -name "*.log" -mtime +10 -exec rm -f {} \;
    echo "Delete old logfiles: ${logfiles}, ret: $?"
fi
```
再配置crontab
```
echo '0 0 * * * (cd /path/to; delete_old_logfile.sh)' >> /var/spool/cron/root
```

## 如何测试
在/data/log目录下手动创建几个.log文件，用touch命令把文件的mtime改到10天前
```
touch -m -d "1999-01-01 00:00:00" /data/log/*.log
```
手动修改系统时间到23:59:50, 观察0点钟crontab定时任务是否执行
```
date -s 23:59:50
```

## 参考
[Linux文件的三个时间](https://www.cnblogs.com/renshengdezheli/p/13941084.html)
