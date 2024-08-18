---
layout: next
title: Python保留最后N个元素
date: 2023-08-29 21:56:54
categories: Python
tags: Python
---

使用`deque`，指定`maxlen`参数的值为N，例如：
```python
>>> from collections import deque
>>> dq = deque(maxlen=3)
>>> dq.append(1)
>>> dq.append(2)
>>> dq.append(3)
>>> dq.append(4)
>>> dq.append(5)
>>> print(dq)
deque([3, 4, 5], maxlen=3)
```

## 参考
[Python Cookbook 1.3](https://python3-cookbook.readthedocs.io/zh_CN/latest/c01/p03_keep_last_n_items.html)
