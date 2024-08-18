---
layout: next
title: Effective Python 笔记
date: 2023-08-12 21:51:51
categories: Python
tags: Python
---

# 前言
不错的Python进阶书。写这个笔记的目的是，把书中提到的编写高质量Python的59个方法的要点记录下来，便于今后的工作中查阅

# 第1章：用Pythonic方式来思考
<!-- more -->

## &#x20;1. 用Pythonic方式来思考

python之禅 —— `import this`

## &#x20;2. 遵循PEP8风格

PEP8 —— Python Enhancement Proposal #8

空白：

*   使用空格而不是tab表示缩进
*   和语法相关每一层缩进用4个空格
*   每行字符数不超过79
*   占据多行表达式，除首行外其余各行应在通常缩进级别上加4空格
*   函数与类之间两个空行隔开
*   同一个类中，各方法一个空行隔开
*   使用下标获取元素，不要在两旁加空格
*   为变量赋值，赋值符号左侧和右侧各自写一个空格，不要在两旁添加空格

命名：

*   函数、变量、属性用小写，各单词以下划线连接
*   受保护实例属性，以单下划线开头
*   私有实例属性：以两个下划线开头
*   类以异常，应每个单词以大写形式命名，如CapitalizedWord
*   模块级别常量，应全部采用大写，各单词以下划线相连，如ALL\_CAPS
*   类中实例方法，应该把首个参数命名为self，以表示对象自身
*   类方法首个参数，应该命名为cls，以表示类自身

表达式和语句：

*   采用内联形式否定词，而不是把否定词放在整个表达式前面，例如写if a is not b 而不是 if not a is b
*   不要通过测长度方法判断list是否为空，应采用if not list，空值会自动评估为False
*   import语句总应放在文件开头
*   引入模块时总应使用绝对名称，不应根据当前模块路径使用相对名称，例如引入bar包中的foo模块，应完整写出from bar import foo，不应该简写import foo
*   文件中import语句应按顺序划三部分，分别表示标准库模块，第三方模块，自用模块，各部分中的import语句应按模块的字母顺序排列

## 3. 了解bytes, str, unicode区别

*   Python3中，bytes是一个包含8位置的序列，str是一种包含unicode字符序列

## 4. 用辅助函数取代复杂表达式

开发者容易过度运用Python语法特性，写出特别复杂难以理解的单行表达式。**这种情况需要把复杂表达式移到辅助函数**

## 5. 了解切割序列的方法

```python
a = ['a','b','c','d','e','f','g','h']
>>> a[:4]
['a', 'b', 'c', 'd']
>>> a[5:]
['f', 'g', 'h']
>>> a[-2:]
['g', 'h']
>>> a[2:-3]
['c', 'd', 'e']
>>> a[:]
['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h']
a[2:6] = ['c', 'a', 'o']
>>> ['a', 'b', 'c', 'a', 'o', 'g', 'h']
```

## 6. 单次切片操作内，不要同时指定start, end, stride

除了基本切片操作，Python还提供了somelist\[start\:end\:stride]形式写法，以实现步进式切割，就是从每n个元素里取1个出来

既有start, end，又有stride的切割操作，可能令人费解，考虑拆解两条语句，其中一条做范围切割，另一条做步进切割，如下

```python
b = a[::2]    # ['a', 'c', 'e', 'g']
c = b[1:-1]   # ['c', 'e']
```

## &#x20;7. 用列表推导取代map, filter

```python
a = [1, 2, 3]
squares = [ x**2 for x in a ]
>>> print(squares)
[1, 4, 9]
或者map, filter
squares = map(lambda x: x ** 2, a)
>>> list(squares)
[1, 4, 9]
```

## 8. 不要使用含有两个以上表达式的列表推导

例：

```python
my_lists = [
    [[1,2,3], [4,5,6]],
]
flat = [x for sublist1 in my_lists
		  for sublist2 in sublist1
		  for x in sublist2]

# 可以看出，列表推导写法不够简洁，改用普通循环语句实现相同效果
flat = []
for sublist1 in my_lists:
	for sublist2 in sublist1:
		flat.extend(sublist2)
```

## 9. 用生成器表达式改写数据量较大的列表推导

```python
it = (len(x) for x in open('file.txt')
print(it)
>>>
generator obj at XXX
print(next(it))
```

## 10. 尽量用enumerate取代range

```python
flavor_list = ['vanilla', 'chocolate', 'pecan', 'strawberry']
for i, flavor in enumerate(flavor_list):
	print('%d: %s' % (i + 1, flavor))
```

还可以直接指定enumerate函数开始计数时所用值，这样能把代码写的更短

```python
for i, flavor in enumerate(flavor_list, 1):
	print('%d: %s' % (i, flavor))
```

## 11. 用zip函数同时遍历两个迭代器

使用range或enumerate方法的代码不够简洁

```python
names = ['Cecilia', 'Lise', 'Marie']
letters = [ len(s) for s in names]

longest_name = None
max_letters = 0
for i, name in enumerate(names):
    if letters[i] > max_letters:
        max_letters = letters[i]
        longest_name = name

print('longest_name: ', longest_name)
```

改用内置zip方法，使代码更清晰

```python
names = ['Cecilia', 'Lise', 'Marie']
letters = [ len(s) for s in names]

longest_name = None
max_letters = 0

for name, count in zip(names, letters):
    if count > max_letters:
        longest_name = name
        max_letters = count

print('longest_name: ', longest_name)
```

注意点：

*   Python2中zip不是生成器，而是会把开发者提供那些迭代器都平行遍历一次，可能占用大量内存导致程序崩溃
*   如果输入迭代器长度不同，zip会提前终止，将导致意外结果

## 12. 不要在for和while循环后写else块

```python
for i in range(3):
	print('Loop %d' % i)
else:
	print('Else block!')
```

*   else块会在整个循环执行完之后立刻运行

```python
for i in range(3):
    print('Loop %d' % i)
else:
    print('Else block!')

$ python test.py
Loop 0
Loop 1
Loop 2
Else block!
```

*   循环里用break提前跳出，会导致else块不执行

```python
for i in range(3):
	print('Loop %d' % i)
	if i == 1:
		break
else:
	print('Else block!')
>>>
Loop 0
Loop 1
```

*   如果for循环遍历序列是空的，就会立刻执行else块

```python
for x in []:
	print('Never runs')
else:
	print('For Else block!')
>>>
For Else block!
```

要点：不要在循环后使用else块，这种写法即不直观，又容易让人误解

## 13. 合理利用try/except/else/finally结构中每个代码块

*   无论try是否异常，都可利用try/finally复合语句中finally块执行清理工作
*   else块可用来缩减try代码量，并把没有发生异常所要执行的语句与try/except分开
*   顺利运行try块后，若想使某些操作能在finaly块的清理代码前执行，可将这些操作写到else

```python
UNDEFINED = object()
def divide_json(path):
	handle = open(path, 'r+')
	try:
		data = handle.read()
		op = json.loads(data)
		value = (op['numerator']/op['denominator'])
	except ZeroDivisionError es e:
		return UNDEFINED
    else:
	    op['result'] = value
	    result = json.dumps(op)
	    handle.seek(0)
	    handle.write(result)
	    return value
    finally:
	    handle.close()
```

# 第2章 函数

## 14. 尽量用异常来表示特殊情况，而不返回Nones

例：令函数返回None，可能会使调用该函数人犯错

```python
def divide(a, b):
	try:
		return a / b
	except ZeroDivisionError:
		return None

x,y = 0, 5
result = divide(x, y)
if not result:
	print('Invaid inputs') # This is WRONG !
```

改进方法一： 把返回值拆两部分，例：

```python
def divide(a, b):
	try:
		return True, a / b
	except ZeroDivisionError:
		return False, None
```

改进方法二：给上一级抛异常

```python
def divide(a, b):
	try:
		return a / b:
	except ZeroDivisionError as e:
		raise ValueError('Invalid inputs') from e
```

## 15. 了解如何在闭包里使用外围作用域中的变量

使用nonlocal获取闭包内的数据

```python
def sort_priority3(numbers, group):
	found = False
	def helper(x):
		nonlocal found
		if x in group:
			found = True
			return (0, x)
		return (1, x)
	numbers.sort(key=helper)
	return found
```

## 16. 用生成器改写直接返回列表函数

这种写法的问题是，返回前要把所有结果放在列表里，如果输入量非常大，程序可能耗尽内存崩溃

```python
def index_words(text):
	result = []
	if text:
		result.append(0)
	for index, letter in enumerate(text):
		if letter == ' ':
			result.append(index + 1)
	return result

address = 'Four score and seven years ago...'
result = index_words(address)
print(result[:3])
>>>
[0, 5, 11]
```

用生成器改写会更好，生成器用yield表达式函数，调用生成器函数时，不会真的运行，而是返回迭代器

```python
def index_word_iter(text):
	if text:
		yield 0
	for index, letter in enumerate(text):
		if letter == ' ':
			yield index + 1

address = 'Four score and seven years ago...'
result = list(index_words_iter(address))
[0, 5, 11, 15, 21, 27]
```

## 17. 在参数上迭代时，多加小心

函数在输入参数上多次迭代时要当心，如果参数是迭代器，可能导致奇怪的行为并错失某些值

## 18. 用数量可变位置参数减少视觉杂讯 (visual noise)

```python
def log(message, *values):
	if not values:
		print(message)
	else:
		values_str = ', '.join(str(x) for x in values)
		print('%s: %s' % (message, values_str))

log('My numbers are', 1, 2)
log('Hi there')
```

要点：

*   def语句中使用\*args, 即可令函数接收数量可变的位置参数
*   在已经接收\*args参数的函数上继续添加位置参数，可能产生难以排查的BUG

## 19. 用关键字参数表达可选的行为

```python
def remainder(number, divisor):
	return number % divisor

>>> 下面这些调用，都是等效的
remainder(20, 7)
remainder(20, divisor=7)
remainder(number=20, divisor=7)
remainder(ivisor=7, number=20)
```

灵活使用关键字参数，提供如下好处：

*   更容易理解含义
*   在函数定义中提供默认值，使代码简洁

## 20. 用None和文档字符串描述含有动态默认值的参数

参数默认值，只会在程序加载模块并读到本函数的定义时评估一次，对于{}或\[]等动态的值，这可能会导致奇怪的行为

```python
def decode(data, default={}):
	try:
		return json.loads(data)
	except ValueError:
		return default

foo = decode('bad data')
foo['stuff'] = 5
bar = decode('also bad')
bar['meep'] = 1
>>>
Foo: {'stuff': 5, 'meep': 1}
Bar: {'stuff': 5, 'meep': 1} # Wrong
```

解决方法是，把关键字参数默认设置为None，并在文档字符串中描述它的行为

```python
def decode(data, default=None):
	""" Load JSON data from a string

	Args:
		data: JSON data to decode
		default: Value to return if decoding fails, Defaults to an empty dictionary
	"""
    if default is None:
        default = {}
    try:
        return json.loads(data)
    except ValueError:
        return default
```

## 21. 用只能以关键字形式指定的参数来确保代码明晰

# 第3章 类与继承

## 22. 尽量用辅助类维护程序状态，而不要用字典和元组

*   不要使用包含其他字典的字典，也不要使用过长的元组
*   如果容器中包含简单而又不可变的数据，可先使用namedtuple表示，后有需要再修改为类
*   保存内部状态的字典如果比较复杂，应该把代码拆解为辅助类

## 23. 简单的接口应该接受函数，而不是类的实例

Python有许多内置API，允许调用者传入函数，以定制其行为，例：

```python
>>> names = ['Socrates', 'Archimedes', 'Plato', 'Aristotle']
>>> names.sort(key=lambda x: len(x))
>>> print(names)
['Plato', 'Socrates', 'Aristotle', 'Archimedes']
```

其他编程语言可能用抽象类定义挂钩，然而在Python中，很多挂钩只是无状态函数，这些函数有明确参数和返回值。用函数做挂钩，比定义一个类要简单

## 24. 以@classmethod形式的多态去通用地构建对象

通过@classmethod机制，可以用一种与构造器相仿的方式构造类的对象

## 25. 用super初始化父类

*   直接调用父类的\_\_init\_\_方法，在多重继承影响下，可能产生无法预知行为
*   总是应该使用内置的super函数初始化父类

## 26. 只在使用Mix-in组件制作工具类时进行多重继承

## 27. 多用public属性，少用private属性

*   Python编译器无法严格保证private字段私密性，用经典的格言来说：“we are all consenting adults here”，大家都认为开放比封闭更好
*   只有子类不受自己控制时，才可以考虑用private属性来避免名称冲突

## 28. 继承collections.abc以实现自定义的容器类型

编写自制容器类型，可以从内置collections.abc模块的抽象基类中继承。从这样的基类继承了子类后，如果忘记实现某个方法，那么collections.abc模块会指出这个错误：

    from collections.abc import Sequence
    class BadType(Sequence):
    	pass
    foo = BadType()
    >>>
    TypeError: Can't instantiate abstract class BadType with ...

# 第4章 元素与属性

## 29. 用纯属性取代get和set方法

从其他语言转入Python的开发者，可能会在类中明确实现getter, setter，但这种做法不像Python编程风格

```python
class OldResistor(object):
    def __init__(self, ohms):
        self._ohms = ohms
    def get_ohms(self):
        return self._ohms
    def set_ohms(self, ohms):
        self._ohms = ohms


obj = OldResistor(50e3)
print(obj.get_ohms())
```

但是，对Python来说，不需要手工实现setter和getter，而是直接用public属性，这样原地自增操作变得清晰自然

```python
class Resistor(object):
	def __init(self, ohms):
		self.ohms = ohms
r1 = Resistor(50e3)
r1.ohms += 5e3
```

## 30. 考虑用@property代替属性重构

Python内置的@property装饰器负责把一个方法变成属性调用

例，使用@property既能检查参数，又可以用类似属性这样简单方法访问类的变量

```python
class Student(object):
    @property
    def score(self):
        return self.score

    @score.setter
    def score(self, value):
        if not isinstance(value, int):
            raise ValueError('score must be an int!')
        if value < 0 or value > 100:
            raise ValueError('score must between 0 ~ 100')
        self._score = value

s = Student()
s.score = 60
s.score = 10000
>>> python test.py
ValueError: score must between 0 ~ 100
```

## 31. 用描述符来改写需要复用的@property方法

## 32. 用getattr, getattribute, setattr实现按需生成操作 (惰性方式加载并保存对象属性）

*   如果某个类定义了\_\_getattr\_\_，同时系统在该类对象实例字典中找不到待查属性，那么，系统就会调用这个方法
*   程序每次访问对象属性，Python系统会调用\_\_getattribute\_\_

## 33. 用元类验证子类

## 34. 用元类注册子类

# 第5章 并发及并行

## 36. 用subprocess模块管理子进程

```python
import subprocess

proc = subprocess.Popen(['echo', 'hello'], stdout=subprocess.PIPE)
out, err = proc.communicate()
print(out.decode('utf-8'))
```

把子进程从父进程中剥离

```python
import subprocess
import time


def run_sleep(period):
    proc = subprocess.Popen(['sleep', str(period)])
    return proc

start = time.time()
procs = []
for _ in range(10):
    proc = run_sleep(0.1)
    procs.append(proc)

for proc in procs:
    proc.communicate()
end = time.time()
print('Finished in %.3f seconds' % (end - start))
```

## 37. 可以用线程执行阻塞式IO，但不要用它做并行计算

*   受到全局解释器锁（GIL）限制，多条Python线程不能在多个CPU核心上平行地执行字节码

## 38. 在线程中使用Lock来防止数据竞争

*   GIL不会保护开发者自己代码， 同一时刻固然只能有一个Python线程运行，但这个线程正在操作某个数据结构时，其他线程可能会打断它，也就是说Python解释器来执行两个连续字节码指令时，其他线程可能会中途插进来。

threading.Lock类实现了互斥锁

```python
class LockingCounter(object):
	def __init__(self):
		self.lock = Lock()
		self.count = 0
	def increment(self, offset):
		with self.lock:
			self.count += offset
```

## 39. 用Queue协调各线程之间的工作

```python
import Queue
import threading


q = Queue.Queue(10=

def producer():
    for i in range(10):
        q.put(i)

def consumer():
    for i in range(10):
        print(q.get())

threads = []

t1 = threading.Thread(target=consumer)
t2 = threading.Thread(target=producer)
```

## 40.考虑用协程并发地运行多个函数

*   对于生成器的yield表达式来说，外部代码通过send方法传给生成器的那个值，就是该表达式所具备的值

## &#x20;41. 考虑用concurrent.futures实现平行计算

*   multiprocessing模块可用于实现平行计算，利用multiprocessing模块最恰当做法，是通过内置concurrent.futures模块及其ProcessPoolExecutor类使用它

# 第6章 内置模块

## &#x20;42. 用functools.wraps定义函数修饰器

```python
def trace(func):
    def wrapper(*args, **kwargs):
        result = func(*args, **kwargs)
        print('%s(%r, %r) -> %r' % (func.__name__, args, kwargs, result))
        return result
    return wrapper


@trace
def fibonacci(n):
    if n in (0, 1):
        return n
    return fibonacci(n - 2) + fibonacci(n - 1)


print(fibonacci(3))
```

*   Python为修饰器提供专门语法，使得程序运行时，能用一个函数修改另一个函数

## 43. 考虑以contextlib和with语句改写可复用的try/finally代码

用with语句代替try... finally，例如

```python
with open('/path/to/file', 'r') as f:
	f.read()
```

## 44. 用copyreg实现可靠pickle操作

内置的pickle模块能够将Python对象字节序列转化为字节流，也能把字节反序列化成Python对象

## 45. 使用datatime模块处理本地时间而不是time

*   不要用time模块在不同时区进行转换
*   开发者总是应该先把时间表示成UTC，然后对其转换，最后再转回本地时间

## 46. 使用内置算法和数据结构

*   双向队列 deque
*   有序字典 OrderedDict (按照键插入顺序，保留键值对在字典中次序）
*   优先级队列 heapq
*   二分查找 bisect
*   迭代器有关 itertools

## 47.重视精确度场合，应该使用decimal

Decimal类适合用在对精度要求高，对舍入行为要求严格的场合，例如：涉及货币计算的场合

## 48. 学会安装由Python开发者社区构建的模块&#x20;

*   PyPI (Python Package Index) 包含许多常用软件包， pip命令行工具可以从PyPI中安装软件包

# 第7章 协作开发

## 49. 为每个函数、类和模块编写文档字符串

**为模块编写文档**

*   每个模块应该有顶级的docstring
*   头一行文字，用一句话描述本模块用途，接着一段话包含一些细节信息，把与本模块操作相关内容，告诉模块使用者。
*   还可以在模块的dockstring中，强调本模块里比较重要类和函数，使开发者据此了解该模块用法

举例：

```python
# words.py
#!/usr/bin/env python3
""" Library for XXX...

Testing how words XXX...

Available functions:
 - palindrome: Determine if a word ...
...
"""
```

**为类编写文档**

每一个类应该有docstring，写法与模块级的docstring大致相同，如下：

```python
class Player(object):
	"""Represents a player of the game.

	Subclassses may override ...

	Public attributes:
	- power: ...
	- coins: ...
	"""
```

**为函数编写文档**

函数的docstring，第一行用一句话描述函数功能，接下来用一段话描述具体行为和函数参数。 若函数由返回值，应该在docstring中写明
```python
    def find_anagrams(word, dictionary):
    	"""Find all anagrams for a word.

    	This function only runs as fast as the test for membership in the dictionary container...

    	Args:
    		word: String of the target word.
    		dictionary: Container with all strings that are known to be actual words.

    	Returns:
    		List of anagrams that were found. Empty if none were found.
```
作者的建议：

*   如果函数没有参数，仅有一个简单返回值，那么只需一句话描述该函数就够了
*   如果函数没有返回值，不要在docstring里提及，不要出现return None这种说法
*   如果正常使用中不会抛异常，就不要在docstring提到异常
*   如果函数接收可变参数，应该在文档中描述\*args, \*\*kwargs用途
*   如果函数参数有默认值，应该指出这些默认值

## 50. 用包安排模块，并提供稳固API

在目录中放入名为\_\_init\_\_.py的空文件，就可以采用相对于该目录的路径，引入目录中其他Python文件，例如：

```python
main.py
mypackage/__init__.py
mypackage/utils.py
```

在main.py中引入utils模块

```python
# main.py
from mypackage import utils
```

**as子句可以给引入当前作用域的属性重新起名，解决冲突**

```python
from analysis.utils import inspect as analysis_inspect
from frontend.utils import inspect as frontend_inspect
```

尽量不要使用import \*语句，而是应该用from x import y，明确指出自己想要引入的名称

## 51. 为自编的模块定义根异常，以便将调用者与API隔离

## 52. 用适当方式打破循环依赖关系

*   如果两个模块必须相互调用对方，才能完成引入操作，就会出现循环依赖现象，这可能导致程序启动时崩溃
*   打破循环依赖关系最佳方案，是把导致两个模块互相依赖的那部分代码，重构为单独模块，并把它放在依赖树底部

## 53. 用虚拟环境隔离项目，并重建其依赖关系

# &#x20;第8章 部署

## 54. 考虑用模块级别代码配置不同部署环境&#x20;

## 55. 通过repr字符串输出调试信息

简单的print不能打印出值的类型信息，可以用repr获取类型信息

## 56. 用unittest测试全部代码

## 57. 用pdb实现交互调试

<https://docs.python.org/zh-cn/3/library/pdb.html>

## 58. 先分析性能，然后再优化

## 59. 用tracemalloc掌握内存使用和泄漏情况


