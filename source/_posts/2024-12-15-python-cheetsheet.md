---
layout: next
title: Python Cheetsheet
date: 2024-12-15 17:06:44
categories: Python
tags:
- Python
- cheetsheet
---

整理如下内容:
* 数据结构和算法
* 常用库和使用方法示例
* Flask cheetsheet
<!-- more -->

# 数据结构和算法

## 一组序列，保存最后N个元素
使用deque，指定maxlen参数的值为N，例如：
```py
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

## 查找最大或最小的N个元素
除了直接排序，还可以利用heaq模块的nlargest()和nsmallest()方法，例如：
```
>>> nums = [3, 5, 2, 4, 1]
>>> smallest = heapq.nsmallest(3, nums)
>>> print(smallest)
[1, 2, 3]
>>> largest = heapq.nlargest(3, nums)
>>> print(largest)
[5, 4, 3]
```

# 常用库和使用方法示例

## 引用别的模块Logging
logging_config.py
```python
import logging
import os
 
# 获取当前脚本所在目录的上级目录作为日志目录
log_dir = os.path.join(os.path.dirname(os.path.dirname(os.path.abspath(__file__))), 'logs')
os.makedirs(log_dir, exist_ok=True)
 
# 日志文件路径
log_file = os.path.join(log_dir, 'app.log')
 
# 配置日志
logging.basicConfig(
    level=logging.DEBUG,  # 设置日志级别
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',  # 设置日志格式
    handlers=[
        logging.FileHandler(log_file),  # 将日志写入文件
        logging.StreamHandler()  # 同时将日志输出到控制台（可选）
    ]
)
 
# 获取logger对象
logger = logging.getLogger(__name__)
```
module_a.py
```python
import logging_config  # 导入日志配置模块
 
# 使用配置好的logger
logger = logging_config.logger
 
def some_function():
    logger.info("This is an info message from module_a.")
    # ... 其他代码
 
if __name__ == "__main__":
    some_function()
```
module_b.py
```python
import logging_config  # 同样导入日志配置模块
 
# 使用配置好的logger
logger = logging_config.logger
 
def another_function():
    logger.error("This is an error message from module_b.")
    # ... 其他代码
 
if __name__ == "__main__":
    another_function()
```
在这个例子中，logging_config.py负责配置日志，包括设置日志文件的路径。module_a.py和module_b.py都导入了logging_config模块，并使用其中配置好的logger对象来记录日志。这样，所有的日志都会写入到同一个日志文件中（在这个例子中是logs/app.log），并且日志格式和级别都是一致的。
请注意，这种方法假设你的项目结构允许logging_config.py中的__file__变量正确地定位到日志目录的上级目录。如果你的项目结构不同，你可能需要调整log_dir的计算方式。此外，如果你想要在每个模块中都有自己独立的日志配置（比如不同的日志文件路径或格式），你可能需要为每个模块分别配置日志。

## 用Queue实现生产者和消费者
```
import threading
import queue
import time
import random

# 创建一个队列对象
q = queue.Queue()

# 生产者线程函数
def producer(q, count):
    for i in range(count):
        item = random.randint(1, 100)  # 生产一个随机整数作为产品
        q.put(item)  # 将产品放入队列
        print(f"Produced {item}")
        time.sleep(random.uniform(0.1, 0.5))  # 模拟生产时间

# 消费者线程函数
def consumer(q):
    while True:
        item = q.get()  # 从队列中获取产品
        if item is None:  # 如果获取到的是 None，则停止消费
            break
        print(f"Consumed {item}")
        time.sleep(random.uniform(0.1, 0.5))  # 模拟消费时间
        q.task_done()  # 通知队列任务已完成

# 创建并启动生产者线程
producer_thread = threading.Thread(target=producer, args=(q, 10))
producer_thread.start()

# 创建并启动消费者线程
consumer_thread = threading.Thread(target=consumer, args=(q,))
consumer_thread.start()

# 等待生产者线程完成
producer_thread.join()

# 在生产者完成后，向队列发送一个 None 来通知消费者停止
q.put(None)

# 等待消费者线程完成队列中的所有任务
q.join()

# 注意：在实际应用中，消费者线程可能需要一种更优雅的方式来停止，
# 而不是依赖于队列中的 None 值。这可以通过设置一个全局停止标志或使用其他线程同步机制来实现。
```

# 把list转换为字符串
```
str = ' '.join(list)
```

## 判断变量是否为None
```
if x is not None:
if not (x is None):
```

## 使用global修改全局变量
```
#!/usr/bin/env python3
x = 0
def inc():
    global x
    x = x + 1
inc()
print(x)
```

## 格式化打印字符串
```py
'hello {language}'.format(language='Python')
```

## print打印不换行
```
print('a', end=' ')
print('b', end=' ')
```

## 判断文件是否存在
```py
abspath = '/etc/v1-config/xxx'
if os.path.exists(abspath) == False:
	print('file not exist')
```

## 逐行读取文件内容，记录到数组, 再按原序去重
```py
with open(domains_file, 'r') as f:
    content = f.readlines()
    for line in content:
        domain_list.append(line.strip('\n'))
    directOutDomains = list(set(domain_list)) # de‑duplicate with original order
    directOutDomains.sort(key=domain_list.index)
```

## 批量执行Shell语句
```py
cmd = '''sed -i '$a\\acl direct-out-servers dstdomain {domain_list}' {cfg_path}; \
                     sed -i '$a\\cache_peer 127.0.0.1 parent 8081 0 no-query default login={u}:{p}' {cfg_path}; \
                     sed -i '$a\\cache_peer_access 127.0.0.1 allow !direct-out-servers' {cfg_path}; \
                     sed -i '$a\\cache_peer_access 127.0.0.1 deny all' {cfg_path}; \
                     sed -i '$a\\always_direct allow direct-out-servers' {cfg_path}; \
                     sed -i '$a\\never_direct allow !direct-out-servers' {cfg_path}; \
                  '''.format(domain_list=domain_list_str, cfg_path=SQUID_CONF_PATH, u=generic_proxy.user, p=generic_proxy.passwd)
os.system(cmd)
```

## base64 decoding end encoding
decoding
```py
import base64
proxy_user = base64.b64decode(proxy_user_base64 + '===').decode('ascii', 'ignore')
```
encoding
```py
sample_string = "GeeksForGeeks is the best"
sample_string_bytes = sample_string.encode("ascii")
  
base64_bytes = base64.b64encode(sample_string_bytes)
base64_string = base64_bytes.decode("ascii")
```

## 给切片命名，使代码清晰可读
使用内置的slice函数创建切片，而不是硬编码下标，从而增强代码可读性，例如：
```
>>> ip="<127.0.0.1>"
>>> GET_IP = slice(1,-1)
>>> ip[GET_IP]
'127.0.0.1'
```

## 解压序列，可迭代对象赋值给多个变量
```
>>> year, month, day = [2002, 6, 10]
>>> print(year, month, day)
2002 6 10
```

可以用占位符，丢弃其他的值
```
year, _ , _ = [2002,6,10]
print(year)
2002
```

## 解压可迭代对象赋值给多个变量
```
L = [1,2,3,4,5]
first, *middle, last = grades
print(middle)
>>> [2,3,4]
```

# Flask Cheetsheet

## Flask返回状态码
```
from flask import Flask, Response

app = Flask(__name__)

@app.route('/empty-200')
def empty_200():
    return Response(status=200)

if __name__ == '__main__':
    app.run(debug=True)
```

## Flask返回字符串+状态码
```
from flask import Flask

app = Flask(__name__)

@app.route('/return-string')
def return_string():
    return "Hello, this is a string!", 200

if __name__ == '__main__':
    app.run(debug=True)
```

## Flask返回JSON+状态码
```
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/return-json')
def return_json():
    data = {
        "message": "Hello, this is JSON!",
        "status": "success"
    }
    return jsonify(data), 200

if __name__ == '__main__':
    app.run(debug=True)
```
Flask设置响应体, 响应头, 状态码: https://cloud.tencent.com/developer/article/1546924
