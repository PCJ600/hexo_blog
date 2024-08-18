---
layout: next
title: Python解压序列，可迭代对象赋值给多个变量方法
date: 2023-08-22 21:55:15
categories: Python
tags: Python
---

## 解压序列赋值给多个变量

```python
>>> year, month, day = [2002, 6, 10]
>>> print(year, month, day)
2002 6 10
```

可以用占位符，丢弃其他的值

```python
year, _ , _ = [2002,6,10]
print(year)
2002
```

## 解压可迭代对象赋值给多个变量

<!-- more -->

```python
L = [1,2,3,4,5]
first, *middle, last = grades
print(middle)
>>> [2,3,4]
```

## 参考
[Python Cookbook 第一章](https://python3-cookbook.readthedocs.io/zh_CN/latest/c01/p01_unpack_sequence_into_separate_variables.html)
