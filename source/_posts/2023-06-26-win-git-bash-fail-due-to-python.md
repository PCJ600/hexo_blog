---
layout: next
title: Windows上通过git bash执行python卡住的解决方法
date: 2023-06-26 21:46:41
categories: Python
tags:
- Python
- Win
- Git
---


## 解决方法
编辑 `C:\Program Files\Git\etc\profile.d\aliases.sh`，将`python2.7`改成`python`
![](image1.png)
编辑完成后，重启git bash, 输入python即可

<!-- more -->

## 参考
[https://blog.csdn.net/ofreelander/article/details/112058975](https://blog.csdn.net/ofreelander/article/details/112058975)
