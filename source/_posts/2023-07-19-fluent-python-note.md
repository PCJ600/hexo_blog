---
layout: next
title: 流畅的Python 笔记
date: 2023-07-19 21:48:18
categories: Python
tags: Python
---

## 第一章：Python数据类型

### &#x20;为什么需要特殊方法

<!-- more -->

在类中定义特殊方法的例子：

```python
class MyObj:
	def __init(self):
		self.L = [1,2,3,4]
	def __len__(self):
		return len(self.L)
	def __getitem__(self, index):
		return self.L[index]

o = MyObj()
print('len: ', len(o))
print('L[1]: , o[1])
```

特殊方法存在是为了被Python解释器调用的，自己并不需要调用，比如说不应使用my*obj.\_\_len__()，而应该使用len(myobj)*

### 随机返回某个元素

```python
from random import choice
choice(o)   # o = MyObj()
```

### 使用特殊方法实现一个Vector类

```python
from math import hypot
class Vector:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __abs__(self):
        return hypot(self.x, self.y)

    def __bool__(self):
        return bool(abs(self));

    def __repr__(self):
        return "Vector(%r, %r)" % (self.x, self.y)

    def __add__(self, other):
        x = self.x + other.x
        y = self.y + other.y
        return Vector(x, y)

    def __mul__(self, scalar):
        return Vector(self.x * scalar, self.y * scalar)
```

*   Python有一个内置的函数叫repr，它能把一个对象用字符串形式表达式以便辨认
*   通过add和mul函数，支持+和\*运算
*   默认情况，自定义类实例总为真，除非这个类对bool有自己的实现

### 特殊方法一览

[3. Data model — Python 3.11.4 documentation](https://docs.python.org/3/reference/datamodel.html)

### &#x20;为什么len不是普通方法

为了让Python自带的数据结构可以走后门，abs也是同理，但多亏了它是特殊方法，可以把len用于自定义数据类型

### &#x20;本章小结

*   通过特殊方法，自定义数据类型可以表现得跟内置类型一样，从而写出更具表达力的代码
*   Python对象的基本要求就是它得有合理的字符串表示形式，可以通过repr和str满足这个要求，前者方便调试日志，后者给终端用户看(print)

## 第二章：序列构成的数组

### 学会使用列表推导，增强代码可读性

例：把一个字符串变成Unicode代码

```python
symbols = 'abcdefg'
codes = [ ord(ch) for ch in symbols if ord(s) > 100]
print(codes)
[101,102,103]
```

例：笛卡尔积

```python
colors = ['black', 'white']
sizes = ['S', 'M', 'L']
tshirts = [(color,size) for color in colors for size in sizes ]
print(tshirts)
>>> [('black', 'S'), ('black', 'M'), ('black', 'L'), ('whi
te', 'S'), ('white', 'M'), ('white', 'L')]
```

### &#x20;生成器表达式

生成器表达式可以逐个产生元素，而不是先建立一个完整列表，这种方式更加节省内存

```python
symbols = 'abcdefg'
tuple(ord(ch) for ch in symbols if ord(s) > 100)
```

### 元组不仅是不可变列表，还可用于没有字段名的记录

元组拆包

### 使用collections.namedtuple构建简单类

```python
Card = collections.namedtuple('Card', \['rank', 'list']) beer\_card = Card('7', 'diamonds')
```

### 对象切片

可以用s[a: b: c]形式对a和b之间以c为间隔取值，c的值可以为负，负值意味着反向取值。下面3个例子更直观些

c为负值意味着可以反向取值，举例

```python
>>> s = 'bicycle'
>>> s[::3]
'bye'
>>> s[::-1]
'elcycib'
>>> s[::-2]
'eccb'
```

### 切片操作对序列做修改

```python
l = list(range(10))
>>> l
>>> [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
>>> l[2:5] = [20, 30]
>>> [0, 1, 20, 30, 5, 6, 7, 8, 9]
>>> del l[5:7]
>>> [0, 1, 20, 30, 5, 8, 9]
```

### 对序列使用+和\*

```python
board = [['_'] * 3] * 3
board = [['_','_','_'],['_','_','_'],['_','_','_']]
board[1][2] = '0'
board
[['_','_','0'],['_','_','0'],['_','_','0']]
```

### 序列的增量赋值&#x20;

\+= 背后特殊方法是iadd，如果一个类没实现这种方法就回退一步调用add, 比如

```python
>>> a += b
```

*   如果a实现了iadd，就会调用iadd，a就会就地改动； 如果a没实现iadd，a += b这个表达式效果和a = a + b一样

### 对不可变序列做拼接操作
```python
>>> t = (1,2,3)
>>> id(t)
1737687416128
>>> t *= 2
>>> t
(1, 2, 3, 1, 2, 3)
>>> id(t)
1737687389568
```

*   ID发生了变化，对不可变序列进行重复拼接操作效率低，因为每次都有一个新对象，解释器需把原来对象中元素先复制到新对象，再追加新元素
*   str是一个例外，str做+=，id不一定会变，因为CPython做了优化，str初始化时会留额外的可扩展空间，做增量操作时不一定会复制原有字符串到新位置到这类操作

### 关于+=的谜题

```python
>>> t = (1,2,[30,40])
>>> t[2] += [50,60]
TypeError: 'tuple' obj does not support item assignment
>>> t
(1, 2, [30,40,50,60])
```

教训：不要把可变对象放在元组里，增量赋值并非完整操作

### list.sort方法和内置函数sorted

*   list.sort方法就地排序列表, sorted总是返回一个列表
*   两种方法都有两个可选的关键字参数
    *   reverse 如果被设定为True，降序输出
    *   key，默认用元素自己的值做排序
*   是稳定排序，每次排序结果相对位置固定的
```python
>>> fruits = ['grape', 'raspberry', 'apple', 'banana']
>>> sorted(fruits)
['apple', 'banana', 'grape', 'raspberry']
>>> sorted(fruits, reverse=True)
['raspberry', 'grape', 'banana', 'apple']
>>> sorted(fruits, key=len)
['grape', 'apple', 'banana', 'raspberry']
>>> sorted(fruits, key=len, reverse=True)
['raspberry', 'banana', 'grape', 'apple']
>>> fruits.sort()
>>> fruits
['apple', 'banana', 'grape', 'raspberry']
```
### bisect操作（排序，插入已排序的序列）
插入已排序的序列
```python
>>> a = [1, 3, 4, 6]
>>> import bisect
>>> bisect.insort(a, 5)
>>> print(a)
[1, 3, 4, 5, 6]
```
### array数组
*   如需要一个只包含数字的列表, array.array比list更高效
*   数组支持所有可变序列相关操作，包括pop, insert, extend
*   数组提供从文件读取和存入文件的快速方法，如frombytes, tofile
```python
>>> from array import array
>>> from random import random
>>> floats = array('d', (random() for i in range(10**7)))
>>> floats[-1]
0.5225102971704024
>>> fp = open('floats.bin', 'wb')
>>> floats.tofile(fp)
>>> fp.close()
>>> floats2 = array('d')
>>> fp = open('floats.bin', 'rb')
>>> floats2.fromfile(fp, 10**7)
>>> fp.close()
>>> floats2[-1]
0.5225102971704024
>>> floats2 == floats
True
```

*   tofile和fromfile速度比从文本文件读快60多倍，原因是后者使用内置的float把每一行文字转成浮点数
*   另一个快速序列化数字类型方法是使用pickle，速度和array相当，但pickle可以处理几乎所有的内置数字类型，包含复数、嵌套集合，用户自定义类

### &#x20;内存视图

`memoryview`是一个内置类，能让用户在不复制内容情况下操作同一个数组的不同切片

```python
>>> import array
>>> numbers = array.array('h', [-1, 0, 1])
>>> memv = memoryview(numbers)
>>> len(memv)
3
>>> memv[0]
-1
>>> memv_oct = memv.cast('B')
>>> memv_oct.tolist()
[255, 255, 0, 0, 1, 0]
>>> memv_oct[3] = 4
>>> numbers
array('h', [-1, 1024, 1])
```

### NumPy 和 SciPy

*   NumPy提供高阶数组和矩阵操作, SciPy是基于NumPy的库，提供科学计算有关算法

### 双向队列和其他形式的队列

collections.deque类是一个线程安全，可以快速从两端添加或删除元素的数据类型

```python
>>> from collections import deque
>>> dq = deque(range(10), maxlen=10) ➊
>>> dq
deque([0, 1, 2, 3, 4, 5, 6, 7, 8, 9], maxlen=10)
>>> dq.rotate(3) ➋
>>> dq
deque([7, 8, 9, 0, 1, 2, 3, 4, 5, 6], maxlen=10)
>>> dq.rotate(-4)
>>> dq
deque([1, 2, 3, 4, 5, 6, 7, 8, 9, 0], maxlen=10)
>>> dq.appendleft(-1) ➌
>>> dq
deque([-1, 1, 2, 3, 4, 5, 6, 7, 8, 9], maxlen=10)
>>> dq.extend([11, 22, 33]) ➍
>>> dq
deque([3, 4, 5, 6, 7, 8, 9, 11, 22, 33], maxlen=10)
>>> dq.extendleft([10, 20, 30, 40]) ➎
>>> dq
deque([40, 30, 20, 10, 3, 4, 5, 6, 7, 8], maxlen=10)
```
*   maxlen是一个可选参数，代表队列可以容纳的元素数量，一旦设定不可修改
*   rotate操作接受一个参数n，当n > 0时，最右边n个元素移动到最左，n < 0, 最左边n个元素移动到最右

除了deque，其他Python标准库也有对队列的实现

*   queue
*   multiprocessing 实现了自己的Queue
*   asyncio
*   heapq

### &#x20;sorted, list.sort背后的排序算法 Timsort

Timsort是一种自适应排序算法，会根据原始数据的顺序特点交替用插入排序和归并排序，以达到最佳效率

## 第3章 字典和集合

### 可散列的数据类型

*   如一个对象时可散列的，在这个对象的生命周期中，它的散列值是不变的，而且这个对象需要实现hash方法，另外可散列对象要有qe方法，才能跟其他键做比较
*   原子不可变数据类型(str, bytes, 数值类型）都是可散列类型, frozenset也是可散列的
*   对于元组，只有当一个元组包含所有元素都是可散列类型情况下，它才是可散列的

```python
>>> ttt = (1, 2, (30, 40))
>>> hash(ttt)
-3907003130834322577
>>> t2 = (1, 2, [30, 40])
>>> hash(t2)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: unhashable type: 'list'
```

### &#x20;字典的构造方法

```python
>>> a =  dict(one=1, two=2)
>>> b = {'one': 1, 'two': 2}
>>> c = dict(zip(['one', 'two'], [1,2]))
>>> d = dict([('one',1), ('two',2)])
>>> e = dict({'one':1, 'two':2})
>>> a == b == c == d == e
True
```

### 字典推导

字典推导可以从任何以键值对作为元素可迭代对象中构建出字典

```python
>>> array = [('one', 1), ('two', 2), ('three', 3)]
>>> dic = {k: v for v, k in array}
>>> dic
{1: 'one', 2: 'two', 3: 'three'}
```

### 用setdefault处理找不到的键

以下两种写法是等价的

```python
if key not in my_dict:
	my_dict[key] = []
my_dicy[key].append(new_value)

my_dict.setdefault(key, []).append(new_value)
```

也可以用get方法获得key对应的value，如果不存在返回默认值

    >>> dic = { 1: 'one'}
    >>> dic.get(2, 'N/A')
    'N/A'

### 字典的变种

collections.OrderedDict

添加key时会保持顺序

collections.ChainMap

容纳数个不同映射对象，进行key查找时，这些对象会被当做一个整体被逐个查找

### 不可变映射类型

MappingProxyType

### 集合论

set和它的不可变姊妹类型frozenset，集合中元素必须是可散列的

创建空集合用 set(), frozenset({0,1,2,3})

### 往字典里添加新键可能会改变已有键顺序

不要对字典同时进行迭代和修改，最好分两步：首先对字典迭代，得出要添加内容，把这些内容放在一个新字典里，迭代结束后再对原有字典进行更新

## 第4章 文本和字节序列

*   字符、码位和字节表述&#x20;
*   bytes、bytearray 和 memoryview 等二进制序列的独特特性&#x20;
*   全部 Unicode 和陈旧字符集的编解码器&#x20;
*   避免和处理编码错误&#x20;
*   处理文本文件的最佳实践&#x20;
*   默认编码的陷阱和标准 I/O 的问题&#x20;
*   规范化 Unicode 文本，进行安全的比较&#x20;
*   规范化、大小写折叠和暴力移除音调符号的实用函数&#x20;
*   使用 locale 模块和 PyUCA 库正确地排序 Unicode 文本&#x20;
*   Unicode 数据库中的字符元数据 能处理字符串和字节序列的双模式 API

字符串是一个字符序列，从Python3的str对下井获取元素是Unicode字符，Unicode标准把字符的标识和具体字节表述进行如下明确区分

*   字符标识，即码位，是0-1114111的数字，在Unicode标准中以4-6个十六进制数字表示，加前缀U+, 例如字母A的码位是U+0041，欧元符号码位阿是U+20AC
*   字符具体表述取决于所用编码，UTF-8中, A(U+0041)的码位编码成单个字节\x41, 而在UTF-16LE编码中编成两个字节\x41\x00

```python
>>> s = 'café'
>>> len(s)
4
>>> b = s.encode('utf8')
>>> b
b'caf\xc3\xa9'
>>> len(b)
5
>>> b.decode('utf8')
'café'
```

### 字节概要

Python内置两种基本二进制序列类型： bytes, bytearray

### BOM是什么
字节序标记 byte-order mark
## &#x20;第7章 函数装饰器和闭包

装饰器特性，能把被装饰的函数替换成其他函数， 第二个特性：装饰器在加载模块时立即执行

```python
def deco(func):
	def inner():
		print('running inner()')
    return inner
@deco
def target():
	print('running target()')

target()
running inner()
```

打印函数运行时间

```python
import time
def calc_time(func):
    def wrapper(*args, **kwargs):
        t1 = time.time()
        func(*args, **kwargs)
        t2 = time.time()
        print('func {} costs {}'.format(func.__name__, t2 - t1))
    return wrapper

@calc_time
def add(a, b):
    time.sleep(5)
    return a + b

add(1, 2)
$ python calc.py
func add costs 5.014971017837524
```

### 为什么用闭包

举例：求平均数，每次调用avg，新增一个数并重新计算平均值

实现方法1：缺点是需定义全局变量

```python
import time

total = 0
count = 0

def avg(n):
    global total,count
    total += n
    count += 1
    return total / count

print(avg(10))
print(avg(11))
```

实现方法2：使用类，缺点是麻烦

```python
class Aver():
    def __init__(self):
        self.series = []
    def __call__(self, value):
        self.series.append(value)
        total = sum(self.series)
        return total/len(self.series)

avg = Aver()
print(avg(10))
print(avg(11))
```

实现方法3：用函数，引入nonocal声明

```python
def make_aver():
    count = 0
    total = 0
    def aver(val):
        nonlocal count
        nonlocal total
        count += 1
        total += val
        return total / count
    return aver

avg = make_aver()
print(avg(10))
print(avg(11))
```

### 闭包和匿名函数区别

## 第8章 对象引用、可变性和垃圾回收

### &#x20;默认做浅复制

```python
l1 = [3, [66,55,44], (7,8,9)]
l2 = list(l1)
l1.append(100)
l1[1].remove(55)
>>> l1 = [3, [66,44], (7,8,9), 100]
>>> l2 = [3, [66,44], (7,8,9)]
l2[1] += [33,22]
l2[2] += (10,11)
[3, [66,44,33,22], (7,8,9), 100]
[3, [66,44,33,22], (7,8,9,10,11), 100]
```

*   对可变对象来说，l2\[1]引用的列表， +=运算符就地修改列表，修改在l1\[1]中也有体现
*   对元组来说，+=创建一个新元组，然后重新绑定给变量l2\[2]

### 不要使用可变类型作为参数默认值

### 防御可变参数

如果定义函数接收可变参数，应该谨慎考虑调用方是否期望修改传入参数

例：从TwilightBus下车后，乘客消失了

```python
class Bus:
    def __init__(self, passengers=None):
        if passengers is None:
            self.passengers = []
        else:
            self.passengers = passengers
    def drop(self, name):
        self.passengers.remove(name)


basketball_team = ['Sue', 'Tina', 'Maya']
bus = Bus(basketball_team)
bus.drop('Tina')
print(basketball_team)

>>> ['Sue', 'Maya']
```

### del和垃圾回收

对象不会自行销毁，然而无法得到对象时，可能被当做垃圾回收

del语句删除名称，而不是对象，举例：

```python
import weakref
s1 = {1, 2, 3}
s2 = s1
def bye():
	print('Gone with the wind...')

>>> ender = weakref.finalize(s1, bye)
>>> ender.alive
True
>>> del s1
>>> ender.alive
True
>>> s2 = 'spam'
Gone with the wind...
>>> ender.alive
False
```

### &#x20;弱引用

当对象的饮用数量归零后，垃圾回收程序会把对象销毁，但是，有时需要引用对象，而不让对象存在时间超过所需时间，这经常用在缓存中

弱引用不会增加对象引用数量，引用的目标对象称为所指对象，因此我们说，弱引用不会妨碍所指对象被当做垃圾回收

```python
>>> import weakref
>>> a_set = {0, 1}
>>> wref = weakref.ref(a_set)
>>> wref
<weakref at 0x100637598; to 'set' at 0x100636748>
>>> wref()
{0, 1}
>>> a_set = {2, 3, 4}
>>> wref()
{0, 1}
>>> wref() is None
False
>>> wref() is None
True
```

### &#x20;Python对不可变类型施加的把戏

例：使用另一个元组构建元组，得到的是同一个元组

```python
>>> t1 = (1, 2, 3)
>>> t2 = tuple(t1)
>>> t2 is t1
True
>>> t3 = t1[:]
>>> t3 is t1
True
```

例：字符串字面量可能会创建共享的对象

```python
>>> t1 = (1, 2, 3)
>>> t3 = (1, 2, 3)
>>> t3 is t1
False
>>> s1 = 'ABC'
>>> s2 = 'ABC'
>>> s2 is s1
True
```

## 第9章 符合Python风格的对象

讨论两个概念：

*   如何以及何时使用@classmethod和@staticmethod装饰器
*   Python的私有属性和受保护属性的用法，约定和局限

### classmethod和staticmethod

*   classmethod定义了操作类，而不是操作实例的方法, (参数没有this)
*   staticmethod就是普通函数，不过它碰巧在类的定义中体现 (不太有用）

```python
class Demo:
	@classmethod
	def klassmeth(*args):
		return args
	@staticmethod(*args):
		return args
```

<https://julien.danjou.info/blog/2013/guide-python-static-class-abstract-methods>

### Python的私有属性和受保护的属性

*   \_private\_attrs：两个下划线开头，声明该属性为私有，不能在类地外部被使用或直接访问。 在类内部的方法中使用时 self.\_\_private\_attrs
*   \_\_private\_method：两个下划线开头，声明该方法为私有方法，不能在类地外部调用
