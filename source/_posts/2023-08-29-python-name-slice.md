---
layout: next
title: Python给切片命名，增强可读性
date: 2023-08-29 19:34:19
categories: Python
tags: Python
---

使用内置的`slice`函数创建切片，而不是硬编码下标，从而增强代码可读性，例如：
```python
>>> ip="<127.0.0.1>"
>>> GET_IP = slice(1,-1)
>>> ip[GET_IP]
'127.0.0.1'
```

## 参考
[https://python3-cookbook.readthedocs.io/zh_CN/latest/c01/p11_naming_slice.html](https://python3-cookbook.readthedocs.io/zh_CN/latest/c01/p11_naming_slice.html)
