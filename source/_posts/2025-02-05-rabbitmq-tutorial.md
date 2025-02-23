---
layout: next
title: RabbitMQ入门教程
date: 2025-02-05 20:18:12
categories: RabbitMQ
tags: RabbitMQ
---

## 前言
[Linux安装RabbitMQ](https://pcj600.github.io/2024/1218212631.html) 
[官方教程](https://www.rabbitmq.com/tutorials/tutorial-one-python)

## 1. Hello RabbitMQ
使用Python pika客户端, 写一个简单的生产者和消费者
```
Producer ------> Queue -----> Consumer
```
 
### 编写Producer(send.py)
先连接Broker, 建立connection和channel
```py
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()
```
接着声明一个队列; 在RabbitMQ中，消息不是直接发送到队列，而是先发送到交换机(exchange), 由交换机发送到队列。
```py
channel.queue_declare(queue='hello')
```
使用basic_publish发送消息。 这里使用默认交换机(''), routing_key参数指定为队列名称
```py
channel.basic_publish(exchange='', routing_key='hello', body='Hello World!')
```
<!-- more -->
完整的生产者代码(send.py)如下:
```py
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost', credentials=pika.PlainCredentials('admin', 'V2SG@xdr')))
channel = connection.channel()
channel.queue_declare(queue='hello')
channel.basic_publish(exchange='', routing_key='hello', body='Hello World!')
print(" [x] Sent 'Hello World!'")
connection.close()
```

执行成功后，在后台通过`rabbitmqadmin`查看队列已创建，消息已发送
```bash
# rabbitmqctl list_queues
Listing queues for vhost / ...
name    messages
hello   1
```

### 编写Consumer(receive.py)
和send.py类似，需要先连接Broker，创建Connection和Channel, 再声明队列。
通过basic_consume方法接收消息，定义callback消费消息
```py
#!/usr/bin/env python
import pika, sys, os

def main():
    connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost', credentials=pika.PlainCredentials(your_username, your_password)))
    channel = connection.channel()
    channel.queue_declare(queue='hello')
    def callback(ch, method, properties, body):
        print(f" [x] Received {body}")
		
    channel.basic_consume(queue='hello', on_message_callback=callback, auto_ack=True)
	
    print(' [*] Waiting for messages. To exit press CTRL+C')
    channel.start_consuming()

if __name__ == '__main__':
    try:
        main()
    except KeyboardInterrupt:
        print('Interrupted')
        try:
            sys.exit(0)
        except SystemExit:
            os._exit(0)
```
执行结果
```
python3 receive.py
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received b'Hello World!'
```

## 2. Work Queues(工作队列)
工作队列的概念在web开发中经常用到。 因为短的HTTP请求窗口中难以处理耗时较多的任务。

对Hello RabbitMQ的代码稍作修改，在消费者中通过time.sleep()假设我们在执行一个耗时的任务
生产者(new_task.py)
```
import sys

message = ' '.join(sys.argv[1:]) or "Hello World!"
channel.basic_publish(exchange='',
                      routing_key='hello',
                      body=message)
print(f" [x] Sent {message}")
```
消费者(worker.py)
```
import time

def callback(ch, method, properties, body):
    print(f" [x] Received {body.decode()}")
    time.sleep(body.count(b'.'))
    print(" [x] Done")
```

### Round-robin 分发任务
示意图:
```
                                                       /---------> Consumer 1
Producer -------> Queue --------> Round Robin dispatch
                                                       \---------> Consumer 2
```
开启两个窗口运行消费者代码(worker.py)
```
# shell 1
python worker.py
# => [*] Waiting for messages. To exit press CTRL+C
```
```
# shell 2
python worker.py
# => [*] Waiting for messages. To exit press CTRL+C
```

第三个窗口中, 执行生产者代码(new_task.py), 发布5条消息
```bash
# shell 3
python new_task.py First message.
python new_task.py Second message..
python new_task.py Third message...
python new_task.py Fourth message....
python new_task.py Fifth message.....
```

查看消费结果, 的确是Round-robin分发任务
```bash
# shell 1
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received First message.
 [x] Done
 [x] Received Third message...
 [x] Done
 [x] Received Fifth message.....
 [x] Done
```
```
# shell 2
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received Second message..
 [x] Done
 [x] Received Fourth message....
 [x] Done
```

### 消息确认(message acknowledgments)
为了确保消息不丢失, RabbitMQ支持消息确认机制
* 消费者返回一个ACK，告诉RabbitMQ一个特定消息已经被接收或处理, 并且RabbitMQ可以自由删除它
* 如果一个消费者挂了(连接被关闭T丢失)而没有发送ACK，RabbitMQ就知道这个消息没有被正常处理，将消息重新排队; 如果同时有其他消费者在线，RabbitMQ可以迅速将消息重新交付给另一个消费者, 确保没有信息丢失。

这里移除auto_ack=True参数, 调用basic_ack进行手动消息确认
```
def callback(ch, method, properties, body):
    print(f" [x] Received {body.decode()}")
    time.sleep(body.count(b'.') )
    print(" [x] Done")
    ch.basic_ack(delivery_tag = method.delivery_tag)

channel.basic_consume(queue='hello', on_message_callback=callback)
```
注: 消息确认必须在接收消息的同一通道(channel)上发送。 尝试使用不同的通道进行确认将导致异常。

### 常见错误 —— 忘记消息确认(Forgotten acknowledgments)
忘记调用basic_ack做消息确认是一个常见的错误, 且后果很严重。 RabbitMQ会消耗越来越多的内存，因为它不能释放任何未被锁定的消息。
通过如下命令诊断"忘记消息确认"错误:
```
# rabbitmqctl list_queues name messages_ready messages_unacknowledged
+----------------------------------------------+----------+----------------+-------------------------+
|                     name                     | messages | messages_ready | messages_unacknowledged |
+----------------------------------------------+----------+----------------+-------------------------+
| hello                                        | 3        | 3              | 0                       |
+----------------------------------------------+----------+----------------+-------------------------+
```

### 消息持久性
以上案例中, 如果RabbitMQ服务器停止，消息会丢失。
```bash
systemctl restart rabbitmq-server.service

# rabbitmqctl list_queues name messages_ready messages_unacknowledged
(重启后队列消失了，消息也丢失了)
+----------------------------------------------+----------+----------------+-------------------------+
|                     name                     | messages | messages_ready | messages_unacknowledged |
+----------------------------------------------+----------+----------------+-------------------------+
+----------------------------------------------+----------+----------------+-------------------------+

```
如果我们希望队列在RabbitMQ节点重启后仍然存在，需要同时在生产者和消费者端将队列声明为持久(durable)的，
```py
channel.queue_declare(queue='hello', durable=True)
```
进一步, 如果我们希望消息也是持久的, 需要在basic.publish方法中设置delivery_mode参数为pika.DeliveryMode.Persistent
```py
channel.basic_publish(exchange='', routing_key="task_queue", body=message,
                      properties=pika.BasicProperties(
                         delivery_mode = pika.DeliveryMode.Persistent
                      ))
```
注：将消息标记为持久消息不能完全保证消息不丢失。 RabbitMQ不会对每条消息执行fsync(2)——它可能只是保存到缓存中，而不是真正写入磁盘。

### 公平调度
例如有两个worker，所有奇数消息复杂, 处理很慢，而偶数消息很快处理完了。 Round-Robin策略仍然会均匀分发消息，没有考虑worker的负载。
这里采用一种公平调度的策略, 通过`basic_qos`方法, 告诉RabbitMQ不要一次给某一个worker发太多的消息; 如果某个worker正在处理前一条消息, 还没有发送确认，就不要给这个worker再发消息。
```py
channel.basic_qos(prefetch_count=1)
```

完整代码如下:
new_task.py
```py
import pika, sys

connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost', credentials=pika.PlainCredentials(your_username, your_password)))

channel = connection.channel()

channel.queue_declare(queue='task_queue', durable=True)

message = ' '.join(sys.argv[1:]) or "Hello World!"
channel.basic_publish(
    exchange='',
    routing_key='task_queue',
    body=message,
    properties=pika.BasicProperties(
        delivery_mode=pika.DeliveryMode.Persistent
    ))
print(f" [x] Sent {message}")
connection.close()
```
worker.py
```
import pika
import time

connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost', credentials=pika.PlainCredentials(your_username, your_password)))
channel = connection.channel()

channel.queue_declare(queue='task_queue', durable=True)
print(' [*] Waiting for messages. To exit press CTRL+C')


def callback(ch, method, properties, body):
    print(f" [x] Received {body.decode()}")
    time.sleep(body.count(b'.'))
    print(" [x] Done")
    ch.basic_ack(delivery_tag=method.delivery_tag)


channel.basic_qos(prefetch_count=1)
channel.basic_consume(queue='task_queue', on_message_callback=callback)

channel.start_consuming()
```

验证: 发多条消息，其中第一条消息设置很长的耗时，让consumer 1处理, 紧接着再发几条耗时较短消息。
```
producer 1
# python new_task.py First message........................................................
 [x] Sent First message........................................................
# python new_task.py Second message..
 [x] Sent Second message..
# python new_task.py Third message...
```
可以观察到RabbitMQ把除了第一条的消息都传给了consumer 2处理, 使用了fair dispatch策略
```
consumer 1
[x] Received First message........................................................
[x] Done
```
```
consumer 2
 [x] Received Second message..
 [x] Done
 [x] Received Third message...
 [x] Done
```

## 发布/订阅(Publish/Subscribe)
之前的例子中，每个任务只会发送给一个消费者。 这一届介绍如何向多个消费者传递消息，这种模式称为发布订阅

示例: 一个简单的日志系统，一个生产者发布日志消息, 广播给所有消费者, 消费者接收并打印日志
### Exchanges(交换机)
RabbitMQ消息模型的核心思想是: 发布者从不直接把消息发送到某个队列; 发布者只能把消息发送到某个交换机(Exchange)。
交换机的概念很简单: 一边从生产者接收消息，另一边把消息发送给队列
```
                                /--------------> Queue1
Producer ------------> Exchange ---------------> Queue2
                                \--------------> Queue3
```
交换机有四种类型：direct, topic, fanout, headers(不常用)

这里使用exchange_declare声明一个fanout类型交换机, 给交换机起名为`logs`。 fanout类型交换机很容易理解，就是把所有消息广播给这个交换机"认识"的所有队列。
```
channel.exchange_declare(exchange='logs', exchange_type='fanout')
```
修改basic_publish的exchange参数, 把消息发送到fanout交换机上
```
channel.basic_publish(exchange='logs', routing_key='', body=message)
```
使用如下命令查看交换机
```
rabbitmqctl list_exchanges
```
### Temporary Queues(临时队列)
如下方法创建一个临时队列
```py
result = channel.queue_declare(queue="")
# result.method.queue包含一个随机的队列名，例如:`amq.gen-JzTY20BRgKO-HjmUJj0wLg`
```
这个例子中，我们希望当消费者连接断开时, 队列随即被删除。 需要在声明临时队列时指定exclusive参数为True
```py
result = channel.queue_declare(queue='', exclusive=True)
```

### Bindings(绑定交换机和队列)
我们创建了一个fanout交换机和一个队列。现在需要让交换机发送消息给队列。 这种交换机和队列之间的关系叫做绑定(Binding)
```
Producer ----------> Exchange -------binding -------> Queue1
                             \-------binding -------> Queue2
```
使用queue_bind方法，把交换机和队列绑定
```py
channel.queue_bind(exchange='logs', queue=result.method.queue)
```
使用如下命令查看绑定
```bash
rabbitmqctl list_bindings
```

完整的生产者代码(emit_log.py)如下:
```py
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='logs', exchange_type='fanout')

message = ' '.join(sys.argv[1:]) or "info: Hello World!"
channel.basic_publish(exchange='logs', routing_key='', body=message)
print(f" [x] Sent {message}")
connection.close()
```
注: basic_publish中需要指定routing_key, 这里指定为空即可, fanout类型交换机会忽略routing_key的值

完整的消费者代码(receive_logs.py)：
```
#!/usr/bin/env python
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='logs', exchange_type='fanout')

result = channel.queue_declare(queue='', exclusive=True)
queue_name = result.method.queue

channel.queue_bind(exchange='logs', queue=queue_name)

print(' [*] Waiting for logs. To exit press CTRL+C')

def callback(ch, method, properties, body):
    print(f" [x] {body}")

channel.basic_consume(
    queue=queue_name, on_message_callback=callback, auto_ack=True)

channel.start_consuming()
```
开多个窗口，启动多个消费者
```
[consumer 1 to n]
python receive_logs.py
```
启动生产者，发送日志
```
python emit_log.py
 [x] Sent info: Hello World!
```

查看所有消费者都收到了同样的日志消息
```
[consumer 1 to n]
# python3 receive_logs.py
 [*] Waiting for logs. To exit press CTRL+C
 [x] b'info: Hello World!'
 [x] b'info: Hello World!'
```
查看创建的交换机, 队列, 和绑定信息
```
# rabbitmqctl list_exchanges
Listing exchanges for vhost / ...
name    type
logs    fanout
...

# rabbitmqctl list_queues
Listing queues for vhost / ...
name    messages
amq.gen-CbtKVwLyav93M0cG0hBDZg  0

# rabbitmqctl list_bindings
Listing bindings for vhost /...
source_name     source_kind     destination_name        destination_kind        routing_key     arguments
logs    exchange        amq.gen-CbtKVwLyav93M0cG0hBDZg  queue   amq.gen-CbtKVwLyav93M0cG0hBDZg  []
```

## 3. Routing(路由）
https://www.rabbitmq.com/tutorials/tutorial-four-python

上面实现的日志系统采用了fanout类型交换机, 将所有日志广播给消费者。 这一节我们对日志系统进行扩展，根据日志严重程度过滤日志, 改用direct类型交换机。
```                                   
Producer ---------> direct Exchange --------key1-------> Queue 1 -------> Consumer 1
                                    \
                                     \------key2-------> Queue 2 -------> Consumer 2
```
使用direct类型交换机, 需要在queue_bind方法中指定routing_key
```
channel.queue_bind(exchange=exchange_name, queue=queue_name, routing_key='key1')
```

### Multiple Bindings
可以使用相同的routing key绑定多个队列, 例如
```
Producer ---------> direct Exchange	--------key1-------> Queue 1 -------> Consumer 1
                                    \
                                     \------key1-------> Queue 2 -------> Consumer 2
```
此时direct交换机行为和fanout交换机类似, routing_key=key1的消息会同时发送到Queue 1和Queue 2

**发布日志**
首先创建一个direct类型交换机，交换机名称为direct_logs
```py
channel.exchange_declare(exchange='direct_logs', exchange_type='direct')
```
使用basic_publish发送消息, routing_key表示日志等级, 取值为info, warning或error
```py
channel.basic_publish(exchange='direct_logs', routing_key=serverity, body=message)
```
**订阅日志**
声明队列, 创建routing_key到队列的绑定
```
result = channel.queue_declare(queue='', exclusive=True)
queue_name = result.method.queue

for severity in severities:
    channel.queue_bind(exchange='direct_logs',queue=queue_name, routing_key=severity)
```

整个发布订阅流程如下图
```
                                   |--------------error-----------> Queue 1 ---------------> Consumer 1
                                   | 
                                   |    /----------info----------\
Producer -------------> direct Exchange -----------warn-----------> Queue 2 ---------------> Consumer 2
                                        \----------error---------/
```

生产者(emit_log_direct.py)
```py
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='direct_logs', exchange_type='direct')

severity = sys.argv[1] if len(sys.argv) > 1 else 'info'
message = ' '.join(sys.argv[2:]) or 'Hello World!'
channel.basic_publish(exchange='direct_logs', routing_key=severity, body=message)
print(f" [x] Sent {severity}:{message}")
connection.close()
```
消费者(receive_logs_direct.py)
```py
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='direct_logs', exchange_type='direct')

result = channel.queue_declare(queue='', exclusive=True)
queue_name = result.method.queue

severities = sys.argv[1:]
if not severities:
    sys.stderr.write("Usage: %s [info] [warning] [error]\n" % sys.argv[0])
    sys.exit(1)

for severity in severities:
    channel.queue_bind(
        exchange='direct_logs', queue=queue_name, routing_key=severity)

print(' [*] Waiting for logs. To exit press CTRL+C')

def callback(ch, method, properties, body):
    print(f" [x] {method.routing_key}:{body}")

channel.basic_consume(queue=queue_name, on_message_callback=callback, auto_ack=True)

channel.start_consuming()
```

启动第一个消费者，只订阅warning和error等级的日志, 存储到磁盘
```
python receive_logs_direct.py warning error > logs_from_rabbit.log
```
启动第二一个消费者，订阅所有等级的日志，打印到终端
```
python receive_logs_direct.py info warning error
```
最后，启动一个生产者，依次发送error, warning, info等级日志
```
python emit_log_direct.py error "Run. Run. Or it will explode."
 [x] Sent error:Run. Run. Or it will explode.
[root@k8s-node2 04_routing]# python emit_log_direct.py warning "warning message"
 [x] Sent warning:warning message
[root@k8s-node2 04_routing]# python emit_log_direct.py info "info message"
 [x] Sent info:info message
```
查看消费情况。 第一个消费者只保存了error和warning日志到文件，第二个消费者打印了所有日志等级的文件
```
Consumer 1
# python receive_logs_direct.py warning error > logs_from_rabbit.log
^CKeyboardInterrupt
# cat logs_from_rabbit.log
 [*] Waiting for logs. To exit press CTRL+C
 [x] error:b'Run. Run. Or it will explode.'
 [x] warning:b'warning message'
```
```
Consumer 2
# python receive_logs_direct.py info warning error
 [*] Waiting for logs. To exit press CTRL+C
 [x] error:b'Run. Run. Or it will explode.'
 [x] warning:b'warning message'
 [x] info:b'info message'
```

## 4. Topic类型交换机
Topic类型交换机将路由键与某个模式匹配，生产者带routingKey, 消费者带模糊的routingKey
* `*`正好匹配一个词，比如`order.*`匹配`order.insert`
* `#`匹配一个或多个词，比如`order.#`匹配`order.insert.common`

```
                                    /----------kern.*---------> Queue1 ----------> Customer 1
Producer ---------> Exchange(Topic)
                                    \--------*.critical-------> Queue2 ----------> Customer 2
```

生产者(emit_log_topic.py)
```py
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='topic_logs', exchange_type='topic')

routing_key = sys.argv[1] if len(sys.argv) > 2 else 'anonymous.info'
message = ' '.join(sys.argv[2:]) or 'Hello World!'
channel.basic_publish(
    exchange='topic_logs', routing_key=routing_key, body=message)
print(f" [x] Sent {routing_key}:{message}")
connection.close()
```
消费者(receive_logs_topic.py)
```py
#!/usr/bin/env python
import pika
import sys

connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost'))
channel = connection.channel()

channel.exchange_declare(exchange='topic_logs', exchange_type='topic')

result = channel.queue_declare('', exclusive=True)
queue_name = result.method.queue

binding_keys = sys.argv[1:]
if not binding_keys:
    sys.stderr.write("Usage: %s [binding_key]...\n" % sys.argv[0])
    sys.exit(1)

for binding_key in binding_keys:
    channel.queue_bind(
        exchange='topic_logs', queue=queue_name, routing_key=binding_key)

print(' [*] Waiting for logs. To exit press CTRL+C')


def callback(ch, method, properties, body):
    print(f" [x] {method.routing_key}:{body}")

channel.basic_consume(
    queue=queue_name, on_message_callback=callback, auto_ack=True)

channel.start_consuming()
```

接收所有日志
```
python receive_logs_topic.py "#"
```
接收kern模块的日志
```
python receive_logs_topic.py "kern.*"
```
接收critical等级的日志
```
python receive_logs_topic.py "*.critical"
```
接收kern模块或者critical等级日志
```
python receive_logs_topic.py "kern.*" "*.critical"
```

生产者发送消息
```
python emit_log_topic.py "kern.critical" "A critical kernel error"
```

## 参考
【1】 [https://www.rabbitmq.com/tutorials/tutorial-one-python](https://www.rabbitmq.com/tutorials/tutorial-one-python)