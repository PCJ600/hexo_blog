---
layout: next
title: Python 查找最大或最小的N个元素
date: 2023-08-22 19:31:50
categories: Python
tags: Python
---

除了直接排序，还可以利用`heaq`模块的`nlargest()`和`nsmallest()`方法，例如：

```python
>>> nums = [3, 5, 2, 4, 1]
>>> smallest = heapq.nsmallest(3, nums)
>>> print(smallest)
[1, 2, 3]
>>> largest = heapq.nlargest(3, nums)
>>> print(largest)
[5, 4, 3]
```
