---
layout: next
title: Python cookbook 笔记
date: 2023-12-16 19:39:17
categories: Python
tags: Python
---

## 一、数据结构和算法

### 1. 解压序列赋值给多个变量

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
<!-- more -->

### 2. 解压可迭代对象赋值给多个变量

```python
L = [1,2,3,4,5]
first, *middle, last = grades
print(middle)
>>> [2,3,4]
```

### 3. 保留最后N个元素

使用deque，指定maxlen参数的值为AN

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

### 4. 查找最大或最小的N个元素

利用heaq模块的nlargest()和nsmallest()方法

```python
>>> nums = [3, 5, 2, 4, 1]
>>> smallest = heapq.nsmallest(3, nums)
>>> print(smallest)
[1, 2, 3]
>>> largest = heapq.nlargest(3, nums)
>>> print(largest)
[5, 4, 3]
```

### &#x20;5. 字典中的键如何映射多个值

可以将多个值放到列表或集合里。 使用defaultdict自动初始化每个key，只需关注添加元素操作，如下：

```python
>>> from collections import defaultdict
>>> d = defaultdict(list)
>>> d['a'].append(1)
>>> print(d)
defaultdict(<class 'list'>, {'a': [1]})
>>> d = defaultdict(set)
>>> d['a'].add(1)
>>> print(d)
defaultdict(<class 'set'>, {'a': {1}})
```

### 8. 在数据字典中执行计算操作（比如求最小值，最大值，排序等）

考虑下面的股票名和价格

```python
prices = {
	'ACME': 45.23,
	'AAPL': 612.78,
	'IBM': 205.55,
	'HPQ': 37.20,
	'FB': 10.75
}
```

比如查找最小和最大股票价格和股票名称

```python
min_price = min(zip(prices.values(), prices.keys()))
# (10.75, 'FB')
max_price = max(zip(prices.values(), prices.keys()))
# (612.78, 'AAPL')
```

使用zip和sorted函数排列字典数据

```python
prices_sorted = sorted(zip(prices.values(), prices.keys()))
```

### 9. 寻找两个字典的相同点

```python
a = {'x': 1, 'y': 2, 'z': 3}
b = {'w': 10, 'x': 11, 'y': 2}

a.keys() & b.keys() # {'x', 'y'}
a.items() & b.items() # { ('y', 2) }
```

### 以现有字典构造一个排除几个指定键的字典

```python
c = {key: a[key] for key in a.keys() - {'z', 'w'}}
# c is {'x': 1, 'y': 2}
```

### 10.删除序列相同元素并保持顺序

```python
def dedupe(items):
	seen = set()
	for item in items():
		if item not in seen:
			yield item
			seen.add(item)
```

### 11. 给切片命名，是代码清晰可读

利用slice函数，消除代码中的硬编码，使代码清晰可读

```python
>>> ip="<127.0.0.1>"
>>> GET_IP = slice(1,-1)
>>> ip[GET_IP]
'127.0.0.1'
```

### 12. 找出序列中出现次数最多的元素

使用collections.Counter类

```python
words = ['look', 'into', 'into', 'the', 'sky']
from collections import Counter
word_counts = Counter(words)
top_two = word_counts.most_common(2)
python test.py
[('into', 2), ('look', 1)]
```

### 13. 通过某个关键字排序一个字典列表

使用operator模块的`itemgetter`函数

```python
rows = [
{'fname': 'Brian', 'lname': 'Jones', 'uid': 1003},
{'fname': 'David', 'lname': 'Beazley', 'uid': 1002},
{'fname': 'John', 'lname': 'Cleese', 'uid': 1001},
{'fname': 'Big', 'lname': 'Jones', 'uid': 1004}
]
```

根据uid关键字排序

```python
from operator import itemgetter
rows_by_fname = sorted(rows, key=itemgetter('fname'))
rows_by_uid = sorted(rows, key=itemgetter('uid'))
[{'fname': 'John', 'lname': 'Cleese', 'uid': 1001},
{'fname': 'David', 'lname': 'Beazley', 'uid': 1002},
{'fname': 'Brian', 'lname':  'Jones', 'uid': 1003},
{'fname': 'Big', 'lname': 'Jones', 'uid': 1004}]
```

### 14. 排序不支持原生比较的对象

```python
class User:
	def __init__(self, user_id):
		self.user_id = user_id
	def __repr__(self):
		return 'User({})'.format(self.user_id)

users = [User(23),User(3),User(99)]
print(users)
print(sorted(users, key = lambda u: u.user_id))

[User(23), User(3), User(99)]
[User(3), User(23), User(99)]
```

### 15. 通过某个字段将记录分组

```python
from operator import itemgetter
from itertools import groupby

rows = [
{'address': '5412 N CLARK', 'date': '07/01/2012'},
{'address': '5148 N CLARK', 'date': '07/04/2012'},
{'address': '5800 E 58TH', 'date': '07/02/2012'},
{'address': '2122 N CLARK', 'date': '07/03/2012'},
{'address': '5645 N RAVENSWOOD', 'date': '07/02/2012'},
{'address': '1060 W ADDISON', 'date': '07/02/2012'},
{'address': '4801 N BROADWAY', 'date': '07/01/2012'},
{'address': '1039 W GRANVILLE', 'date': '07/04/2012'},
]

rows.sort(key=itemgetter('date'))

for date, items in groupby(rows, key=itemgetter('date')):
    print(date)
    for item in items:
        print(' ', item)
```

运行结果

```python
07/01/2012
  {'address': '5412 N CLARK', 'date': '07/01/2012'}
  {'address': '4801 N BROADWAY', 'date': '07/01/2012'}
07/02/2012
  {'address': '5800 E 58TH', 'date': '07/02/2012'}
  {'address': '5645 N RAVENSWOOD', 'date': '07/02/2012'}
  {'address': '1060 W ADDISON', 'date': '07/02/2012'}
07/03/2012
  {'address': '2122 N CLARK', 'date': '07/03/2012'}
07/04/2012
  {'address': '5148 N CLARK', 'date': '07/04/2012'}
  {'address': '1039 W GRANVILLE', 'date': '07/04/2012'}
```

### 16. 过滤序列元素

最简单的方法是使用列表推导，例如：

```python
>>> mylist = [1, 4, -5, 10, -7, 2, 3, -1]
>>> [n for n in mylist if n > 0]
[1, 4, 10, 2, 3]
>>> [n for n in mylist if n < 0]
[-5, -7, -1]
```

如果对内存敏感，可以用生成器表达式，例如：

```python
>>> pos = (n for n in mylist if n > 0)
>>> for x in pos:
... print(x)
...
1
4
10
2
3
>>>
```

如果过滤规则复杂，不能简单用列表推导，可以考虑用内建的filter()函数，例如：

```python
values = ['1', '2', '-', 'N/A']
def is_int(val):
    try:
        x = int(val)
        return True
    except:
        return False
ivals = list(filter(is_int, values))
print(ivals)
```

如果你需要用另一个相关联的序列来过滤某个序列时，可以考虑itertools.compress()函数，例如：

```python
addresses = [
'5412 N CLARK',
'5148 N CLARK',
'5800 E 58TH',
'2122 N CLARK',
'5645 N RAVENSWOOD',
'1060 W ADDISON',
'4801 N BROADWAY',
'1039 W GRANVILLE',
]
counts = [ 0, 3, 10, 4, 1, 7, 6, 1]

from itertools import compress
more5 = [ n > 5 for n in counts]
result = list(compress(addresses, more5))
print(result)
>>> ['5800 E 58TH', '1060 W ADDISON', '4801 N BROADWAY']
```

### 从字典中提取子集

最简单的方式是使用字典推导， 例如：

```python
prices = {
'ACME': 45.23,
'AAPL': 612.78,
'IBM': 205.55,
'HPQ': 37.20,
'FB': 10.75
}

# Make a dictionary of all prices over 200
p1 = {key: value for key, value in prices.items() if value > 200}
>>> {'AAPL': 612.78, 'IBM': 205.55}
```

## 二、字符串和文本

### 使用多个界定符分割字符串

`re.split()`方法， 可以更加灵活的切割字符串

```python
>>> line = 'asdf fjdk; afed, fjek,asdf, foo'
>>> import re
>>> re.split(r'[;,\s]\s*', line)
['asdf', 'fjdk', 'afed', 'fjek', 'asdf', 'foo']
```

### &#x20;字符串开头或结尾匹配

利用`startswith()`和`endswith()`方法

```python
>>> filename = 'spam.txt'
>>> filename.endswith('.txt')
True
url = 'http://www.python.org'
>>> url.startswith('http:')
True
```

### 用Shell通配符匹配字符串

Python中可以使用`fnmatch()`和`fnmatchcase()`

```python
>>> from fnmatch import fnmatch, fnmatchcase
>>> fnmatch('foo.txt', '*.txt')
True
>>> fnmatch('foo.txt', '?oo.txt')
True
>>> fnmatch('Dat45.csv', 'Dat[0-9]*')
True
```

### 字符串搜索匹配

在字符串中搜索字符串，并返回下标， 使用find()函数

```python
>>> text = 'yeah, but no, but yeah'
>>> text.find('no')
10
>>> text.find('not')
-1
```

对于复杂匹配使用正则表达式和re模块

```python
>>> text1 = '11/27/2012'
>>> text2 = 'Nov 27, 2012'
>>>
>>> import re
>>> # Simple matching: \d+ means match one or more digits
>>> if re.match(r'\d+/\d+/\d+', text1):
... print('yes')
... else:
... print('no')
...
yes
>>> if re.match(r'\d+/\d+/\d+', text2):
... print('yes')
... else:
... print('no')
...
no
>>>
```

注：如果使用同一个模式去多次匹配，应该将模式串预编译为模式对象，比如：

```python
>>> datepat = re.compile(r'\d+/\d+/\d+')
>>> if datepat.match(text1):
... print('yes')
... else:
... print('no')
...
yes
>>> if datepat.match(text2):
... print('yes')
... else:
... print('no')
...
no
>>>
```

### 字符串搜索和替换

对于简单字符串替换，用str.replace()函数

```python
>>> text = 'yeah, but no, but yeah, but no, but yeah'
>>> text.replace('yeah', 'yep')
'yep, but no, but yep, but no, but yep'
```

对于复杂的模式，用re模块的sub函数

```python
>>> text = 'Today is 11/27/2012. PyCon starts 3/13/2013.'
>>> import re
>>> re.sub(r'(\d+)/(\d+)/(\d+)', r'\3-\1-\2', text)
'Today is 2012-11-27. PyCon starts 2013-3-13.'
>>>
```

### &#x20;字符串忽略大小写的搜索替换

使用re模块时添加re.IGNORECASE参数

```python
>>> text = 'UPPER PYTHON, lower python, Mixed Python'
>>> re.findall('python', text, flags=re.IGNORECASE)
['PYTHON', 'python', 'Python']
>>> re.sub('python', 'snake', text, flags=re.IGNORECASE)
'UPPER snake, lower snake, Mixed snake'
>>>
```

### 删除字符串中不需要的字符

strip方法可以删除开头和结尾的空白字符

lstrip()和rstrip()分别从左和从右执行删除操作

除了默认的空白字符，也可以指定其他字符，如下：

```python
>>> s = ' hello world \n'
>>> s.strip()
'hello world'
>>> s.lstrip()
'hello world \n'
>>> s.rstrip()
' hello world'
>>>
>>> # Character stripping
>>> t = '-----hello====='
>>> t.lstrip('-')
'hello====='
>>> t.strip('-=')
'hello'
>>>
```

如果要删除中间的空格，需要用其他技术， 比如replace方法或正则表达式替换

```python
>>> s = ' hello     world \n'
>>> s.replace(' ', '')
'helloworld'
```

### 字符串对齐

基本的字符串对齐操作，可以使用字符串的ljust(),rjust(),center()方法

```python
>>> text = 'Hello World'
>>> text.ljust(20)
'Hello World         '
>>> text.rjust(20)
'         Hello World'
>>> text.center(20)
'    Hello World     '
>>>
>>> text.rjust(20,'=')
'=========Hello World'
>>> text.center(20,'*')
'****Hello World*****'
>>>
```

### 合并字符串

如果待合并的字符串在一个序列中， 最快的方法是使用join()

```python
>>> parts = ['Is', 'Chicago', 'Not', 'Chicago?']
>>> ' '.join(parts)
'Is Chicago Not Chicago?'
>>> '|'.join(parts)
'Is|Chicago|Not|Chicago?'
>>> ''.join(parts)
'IsChicagoNotChicago?'
```

注意不必要的字符串连接操作，比如打印时

```python
print(a + ':' + b + ':' + c) # Ugly
print(':'.join([a, b, c])) # Still ugly
print(a, b, c, sep=':') # Better
```

### 字符串中插入变量

使用format函数

```python
'{name} has {n} message.'.format(name='peter', n=5)
'peter has 5 message.'
```

## 三、数字日期和时间

### 数字四舍五入

简单的舍入运算，使用内置round(value, ndigits)

```python
>>> round(1.23, 1)
1.2
```

格式化输出，使用内置format函数

```python
x = 1234.56789
>>> format(x, '0.2f')
```

### 二八十六进制整数

为了将整数转为二进制，八进制，十六进制文本串。分别使用bin, oct, hex函数

```python
x = 1234
>>> bin(x)
'0b10011010010'
>>> oct(x)
'0o2322'
>>> hex(x)
'0x4d2'
>>> format(x, 'b')
'10011010010'
```

### 随机选择

使用random.choice()， 从一个序列中随机抽取一个元素

```python
>>> import random
>>> values = [1,2,3,4,5,6]
>>> random.sample(values, 3)
[3, 1, 2]
```

随机打乱元素顺序， 使用random.shuffle()

```python
>>> random.shuffle(values)
>>> values
[2, 4, 6, 5, 3, 1]
```

生成随机整数，用random.randint()

```python
>>> random.randint(0,10)
10
```

生成0-1的浮点数

```python
>>> random.random()
0.9406677561675867
```

## 第四章 迭代器与生成器

### 不用for语句迭代

```python
L = [1,2,3,4,5]
it = iter(L)
try:
    while True:
        item = next(it)
        print(item)
except StopIteration:
    pass
```

### 反向迭代

```python
>>> a = [1, 2, 3, 4]
>>> for x in reversed(a):
... print(x)
...
4
3
2
1
```

### 序列上索引值迭代

### 迭代同时，跟踪被处理元素索引

使用内置的`enumerate`函数

```python
>>> my_list = ['a', 'b', 'c']
>>> for idx, val in enumerate(my_list):
... print(idx, val)
...
0 a
1 b
2 c
```

指定开始参数

    >>> my_list = ['a', 'b', 'c']
    >>> for idx, val in enumerate(my_list, 1):
    ... print(idx, val)
    ...
    1 a
    2 b
    3 c

### 同时迭代多个序列

使用zip函数

```python
xpts = [1,2,3,4,5]
ypts = [6,7,8,9,10]

for x,y in zip(xpts,ypts):
    print('x={}, y={}'.format(x, y))
```

zip函数会创建一个迭代器作为结果返回，如果需要将结对值存到列表，使用list函数

```python
>>> zip(a, b)
<zip object at 0x1007001b8>
>>> list(zip(a, b))
[(1, 10), (2, 11), (3, 12)]
```

## 不同集合上元素迭代

场景：想在多个对象执行相同操作， 但对象在不同容器中，希望代码在不失可读性情况下避免写重复循环

使用itertools.chain()方法

```python
>>> from itertools import chain
>>> a = [1, 2, 3, 4]
>>> b = ['x', 'y', 'z']
>>> for x in chain(a, b):
... print(x)
...
1234xyz
>>>
```

## &#x20;顺序迭代合并后排序迭代对象

使用heapq.merge()

```python
>>> import heapq
>>> a = [1, 4, 7, 10]
>>> b = [2, 5, 6, 11]
>>> for c in heapq.merge(a, b):
...     print(c)
...
1
2
4
...
```

`heapq.merge` 可迭代特性意味着它不会立马读取所有序列。 这就意味着你可以在非常长的序列中使用它，而不会有太大的开销。

`heapq.merge()` 需要所有输入序列必须是排过序的。

## 使用其他分隔符或行终止符打印

```python
print('ACME',50,sep=',',end='!!\n')
ACME,50!!
```

输出中禁止换行，用end参数

```python
for i in range(5):
	print(i, end=' ')
0 1 2 3 4 >>>
```

## 函数

### 定义有默认参数的函数

默认参数的值仅仅在函数定义的时候赋值一次

默认参数的值应该是不可变的对象，比如None、True、False、数字或字符串

测试None值时使用 `is` 操作符是很重要的

## 八、类对象

### 改变一个实例字符串表示，可通过重新定义str和repr方法实现：

```python
class Pair:
	def __init__(self, x, y):
		self.x = x
		self.y = y
	def __repr__(self):
		return 'Pair({0.x!r}, {0.y!r})'.format(self)

```
### 类中定义等多个构造器

想实现一个类，除了使用init方法，还有其他方法初始化它

```python
import time
class Date:
""" 方法一：使用类方法"""
# Primary constructor
def __init__(self, year, month, day):
self.year = year
self.month = month
self.day = day
# Alternate constructor
@classmethod
def today(cls):
t = time.localtime()
return cls(t.tm_year, t.tm_mon, t.tm_mday)

a = Date(2012, 12, 21) # Primary
b = Date.today() # Alternate
```

通过字符串调用对象方法

使用attr方法

```python
import math
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    def __repr__(self):
        return 'Point({!r:},{!r:})'.format(self.x, self.y)
    def distance(self, x, y):
        return math.hypot(self.x - x, self.y - y)
p = Point(2, 3)
d = getattr(p, 'distance')(0, 0) # Calls p.distance(0, 0)
```

另一种方法是使用operator.methodcaller()

```python
import operator
operator.methodcaller('distance', 0, 0)(p)
```

## 第十一章 网络与web编程

HTTP请求 urllib.request

创建TCP服务器 socketserver (单线程)

## 第十二章 并发编程

### threading库创建线程
```python
    import time
    def countdown(n):
    	while n > 0:
    		print('T-minus', n)
    		n -= 1
    		time.sleep(5)

    # Create and launch a thread
    from threading import Thread
    t = Thread(target=countdown, args=(10,))
    t.start()
```
### 线程间通信

*   使用Queue对象，Queue对象已包含必要锁，可以用它在多个线程间安全共享数据

```python
from queue import Queue
from threading import Thread
# A thread that produces data
def producer(out_q):
	while True:
	# Produce some data
	...
	out_q.put(data)
# A thread that consumes data
def consumer(in_q):
	while True:
		# Get some data
		data = in_q.get()
		# Process the data
...
# Create the shared queue and launch both threads
q = Queue()
t1 = Thread(target=consumer, args=(q,))
t2 = Thread(target=producer, args=(q,))
t1.start()
t2.start()
```

*   对于生产者消费者速度有差异情况，可以为队列元素添加上限
*   get和put方法支持非阻塞方式和设定超时

### &#x20;给关键代码加锁 —— 使用threading库中的Lock对象

```python
lock = threading.Lock()
with lock:
	XXX
```

## 第十三章 脚本编程与系统管理

```
解析命令行选项
argparse
执行外部命令，获取输出
subprocess.check\_output
复制或启动文件
shutil模块
读取ini配置文件
configparser
```
