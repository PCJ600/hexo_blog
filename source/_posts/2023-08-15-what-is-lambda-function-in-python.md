---
layout: next
title: Python的Lamdba函数是什么，应用场景是什么?
date: 2023-08-15 21:53:10
categories: Python
tags: Python
---


**定义：** Lambda函数也叫匿名函数，它是功能简单，只用一行代码就能实现的小型函数。
**使用场景：** Lambda函数没有名字，不用考虑函数名冲突问题；减少了代码行数，方便又简洁。
**格式：** lambda 参数[,参数] : 表达式 （例： `lambda x,y : x + y`）
**举例：** 用lambda函数求出1到20中所有的奇数并组成一个list：
```python
L = list(filter(lambda x: x % 2 == 1, range(1, 20)))
>>> print(L)
[1, 3, 5, 7, 9, 11, 13, 15, 17, 19]
```
<!-- more -->

另外，匿名函数也可以赋值给变量，可以通过该变量调用这个匿名函数：
```python
f = lambda x: x * x
print(f(5))
25
```

## 参考资料
【1】[https://www.liaoxuefeng.com/wiki/1016959663602400/1017451447842528](https://www.liaoxuefeng.com/wiki/1016959663602400/1017451447842528)
