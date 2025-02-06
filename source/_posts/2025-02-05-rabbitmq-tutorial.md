---
layout: next
title: 消息队列入门 —— RabbitMQ
date: 2025-02-05 20:18:12
tags:
---

https://cloud.tencent.com/developer/article/2113802

# 什么是消息队列

消息队列是在消息传输过程中保存消息的容器。 通常有生产者和消费者两个角色:
* 生产者只负责发送数据到消息队列，谁消费他不管
* 消费者只负责从消息队列中取数据处理，谁发送数据他不管

# 常见消息队列中间件
* Kafka 高吞吐量实时日志采集
* RabbitMQ Erlang语言, 灵活性和易用性，中小规模应用
* RocketMQ 阿里出品, Java开发, 国内市场有很高知名度和应用案例 

# 为什么使用消息队列
<!-- more -->

1. 服务解耦: 通过消息队列, 生产者和消费者之间可以不依赖对方进行通信。 即使某个组件失败，也不影响其他组件，提升可靠性。
```
+------------------+          +------------------+         +------------------+
|   Producer 1     |          |   Producer 2     |         |   Producer N     |
+--------+---------+          +--------+---------+         +--------+---------+
         |                             |                            |
         v                             v                            v
+------------------------------------------------------------------------------+
|                           (Message Queue)                                    |
+------------------------------------------------------------------------------+
         |                             |                            |
         v                             v                            v
+--------+---------+          +--------+---------+          +------------------+
|   Consumer 1     |          |   Consumer 2     |          |   Consumer M     |
+------------------+          +------------------+          +------------------+
```

2. 异步处理: 生产者将请求发送到队列，无需等待请求处理(例如：用户注册发送短信验证码)。 减少响应时间，提升性能和用户体验。 
```
                                                                     /----------------> System B(200ms)
            Request                                                 /
client ------------------->  System A(30ms) --------> Message Queue ------------------> System C(300ms)
       <-------------------									        \
			Response			  					                 \----------------> System D(250ms)

```

3. 流量削峰: 面对高并发请求，通过消息队列缓冲请求，避免系统过载。 确保系统在高峰期稳定运行
```
        10k requests/s                       consume 2k requests/s
User --------------------> Message Queue --------------------------> System A --------------------> MySQL

```

# 消息队列的劣势
* 系统可用性降低，如果MQ挂了，整个系统就崩了
* 系统复杂性增加, 需要考虑消息队列自身的可用性问题，例如：
  * 如何保证消息不被重复消费
  * 如何保证消息可靠传输
  * 如何保证数据一致性
  * 如何解决消息积压导致的故障

# 不适用消息队列的场景
* 对于简单的，快速执行的任务，直接调用API更为直接高效
* 需要立即确认结果的交互
* ...

# RabbitMQ简介
RabbitMQ是实现了高级消息队列协议（AMQP）的开源消息代理软件。RabbitMQ服务器是用Erlang语言编写的

# AMQP是什么
Module Layer: 协议最高层，主要定义了一些客户端调用的命令，客户端可以用这些命令实现自己的业务逻辑。
Session Layer: 中间层，主要负责客户端命令发送给服务器，再将服务端应答返回客户端，提供可靠性同步机制和错误处理。
TransportLayer: 最底层，主要传输二进制数据流，提供帧的处理、信道服用、错误检测和数据表示等。

# RabbitMQ的几个概念
* Broker, RabbitMQ的服务节点，一个broker可以看做一个RabbitMQ服务器
* Exchange(交换机）, 生产者把消息发送到交换机, 交换机负责把消息路由到队列中
* Queue(队列), 用于存储消息。 多个消费者可以订阅同一个队列，此时队列消息会平摊(Round-Robin)给多个消费者处理

# RoutingKey的概念
生产者将消息发送给交换机时，需要指定一个RoutingKey, 用于指定消息的路由规则。

# Binding
通过绑定将交换机和队列关联起来

# RabbitMQ交换机的概念，为什么设计
RabbitMQ中的消息不是直接发送到队列，而是发送到交换机，由交换机发送到队列

# RabbitMQ交换机的几种类型
* fanout 把消息路由到所有与交换机绑定的队列中
* direct 消息路由到BindingKey和RoutingKey完全匹配的队列中
* topic
	* RoutingKey为一个点号分隔的字符串
	* BindingKey也是一个点号分隔的字符串, 可以使用*和#作模糊匹配, *匹配一个单词, #匹配0个或多个
* headers 不常用

# 生产者发送消息流程
* 生产者连接到Broker, 建立连接, 开启一个信道(channel)
* 声明一个交换机
* 声明一些队列
* 通过路由键将交换机和队列绑定起来
* 生产者发送消息到Broker, 其中包含路由键(Routing Key), 交换机等信息
* 交换机根据路由键查找匹配队列，将消息存入响应队列
* RabbitMQ收到确认，从队列中删除已确认的消息
* 关闭信道和连接

# 消费者接受消息
* 消费者连接到Broker，建立连接，开启一个信道（channel)
* 向Broker请求消费队列中的消息, 设置回调函数
* 等待Broker回应，接收信息, 确认收到的消息
* RabbitMQ收到确认，从队列中删除已确认的消息
* 关闭信道和连接

# 交换机无法找到队列时，怎么处理
* mandatory: true, 返回消息给生产者
* mandatory: false, 直接丢弃

# 死信队列是什么
[TODO] 为什么设计？
* 当消息在一个队列中变成dead message之后，可以被重新发送到另一个交换机中，这个交换机就是DLX(Dead-Letter-Exchange)
* 绑定DLX的队列就称为死信队列

# 导致dead message的原因
* 消息被拒
* 消息TTL过期
* 队列满了，无法添加

# 延迟队列
TODO: 为什么设计
当消息发送之后，并不想让消费者立刻拿到消息，而是等待特定时间后，消费者才能拿到这个消息进行消费

# 优先级队列
优先级高的先消费，通过x-max-priority实现，当消费速度比较快，没有消息堆积情况下，这个优先级队列意义不大。

# 事物机制
RabbitMQ客户端中与事务机制相关方法:
* channel.txSelect 用于将当前信道设置为事务模式
* channel.txCommit 提交事务
* channel.txRollback 事务回滚，如果事务提交执行前抛出异常，通过txRollback回滚

# 发送确认机制
生产者把信道设置为confirm模式后，所有在此channel发布消息会被指定一个唯一ID, 
一旦消息被投递到所有匹配队列之后，RabbitMQ就会发送一个确认给生产者，这样生产者就知道消息到达对应的目的地了

# 消息传输保证的层级
* At most once 最多一次，消息可能会丢失，但不会重复传输
* At least once 最少一次，信息不会丢失，但可能重复传输
* Exactly once 恰好一次，消息仅传输一次

# Virtual Host概念
每个RabbitMQ服务器都能创建虚拟的服务器，也叫虚拟主机(vhost)

# 集群中的节点类型
* 内存节点: ram, 将变更写入内存
* 磁盘节点: disc, 磁盘写入操作
RabbitMQ要求最少有一个磁盘节点

# 生产者如何将消息可靠投递到MQ
* Client发送消息给MQ
* MQ将消息持久化后, 发送Ack消息给Client, 如果说因为网络问题导致Ack无法发送到Client, Client在等待超时后会重传消息
* Client收到ACK后，可以认为消息投递成功

# MQ如何把消息可靠投递到消费者
* 消费者收到消息执行业务逻辑
* 发送Ack消息给MQ，通知MQ删除该消息, 此处可能因为网络问题导致ACK失败,导致重复消费，引出消费幂等问题
* MQ将已消费消息删除

# RabbitMQ的高可用
* 单机模式 —— demo, 玩具, 生产环境很少用单机
* 普通集群模式 —— 多台机器启动多个RabbitMQ示例
* 镜像集群模式 —— 高可用模式, 创建的Queue,消息会存在与多个实例上，每次写消息到Queue时，自动把消息到多个实例的Queue进行消息同步


# 如何保证消息的顺序性
* 拆分多个Queue, 每个Queue对应一个Consumer


# 如何保证消息的可靠性
* 生产者到RabbitMQ: Confirm机制 
* RabbitMQ自身: 持久化(交换机和队列), 集群, 普通模式, 镜像模式 (TODO)
* RabbitMQ到消费者: basicAck机制, 死信队列, 消息补偿机制

https://cloud.tencent.com/developer/article/2093640




## Hello RabbitMQ
安装RabbitMQ: []() 
官方教程: [https://www.rabbitmq.com/tutorials/tutorial-one-python](https://www.rabbitmq.com/tutorials/tutorial-one-python)

## 1. Hello World!
使用Python pika客户端, 写一个最基本的生产者和消费者:
```
Producer ------> Queue -----> Consumer
```
 
### 编写Producer(send.py)
首先连接Broker, 建立connection和channel
```
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()
```
接着声明一个队列
```
channel.queue_declare(queue='hello')
```
在RabbitMQ中，消息不是直接发送到队列，而是先发送到交换机(exchange), 由交换机发送到队列。
使用basic_publish发送消息。 这里使用默认交换机(''), routing_key参数指定为队列名称
```
channel.basic_publish(exchange='',
                      routing_key='hello',
                      body='Hello World!')
print(" [x] Sent 'Hello World!'")
```
完整的生产者代码(send.py)如下:
```
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost', credentials=pika.PlainCredentials('admin', 'V2SG@xdr')))
channel = connection.channel()
channel.queue_declare(queue='hello')
channel.basic_publish(exchange='', routing_key='hello', body='Hello World!')
print(" [x] Sent 'Hello World!'")
connection.close()
```

执行成功后，在后台通过`rabbitmqadmin`查看队列已创建，消息已发送
```
rabbitmqadmin --username=XXX --password=YYY list queues name messages
|                     name                     | messages |
+----------------------------------------------+----------+
| hello                                        | 1        |
+----------------------------------------------+----------+
```

### 编写Consumer(receive.py)
和send.py类似，需要先连接Broker，创建Connection和Channel, 再声明队列。
通过basic_consume方法接收消息，定义callback消费消息
```
#!/usr/bin/env python
import pika, sys, os

def main():
    connection = pika.BlockingConnection(pika.ConnectionParameters(host='localhost', credentials=pika.PlainCredentials('admin', 'V2SG@xdr')))
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

## Work queues
工作队列的概念在web开发中经常用到, 因为在短的HTTP请求窗口中难以处理耗时较多的任务。

对上面Hello World的代码稍作修改，在消费者中通过time.sleep()假设我们在执行一个耗时的任务
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
```
# shell 3
python new_task.py First message.
python new_task.py Second message..
python new_task.py Third message...
python new_task.py Fourth message....
python new_task.py Fifth message.....
```

查看消费结果, 的确是Round-robin分发任务
```
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
示意图:
```
                                                       /---------> Consumer 1
Producer -------> Queue --------> Round Robin dispatch
                                                       \---------> Consumer 2
```

### 消息确认
为了确保消息不丢失, RabbitMQ支持消息确认(message acknowledgments)机制
* 消费者返回一个ACK，告诉RabbitMQ一个特定消息已经被接收或处理, 并且RabbitMQ可以自由删除它
* 如果一个消费者挂了(连接被关闭T丢失)而没有发送ACK，RabbitMQ就知道这个消息没有被正常处理，将消息重新排队; 如果同时有其他消费者在线，RabbitMQ可以迅速将消息重新交付给另一个消费者, 确保没有信息丢失。

之前的例子中默认使用自动确认。这里移除auto_ack=True参数, 选择手动消息确认
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
忘记调用`basic_ack`是一个常见的错误, 后果很严重。RabbitMQ会消耗越来越多的内存，因为它不能释放任何未被锁定的消息。

通过如下命令诊断Forgotten acknowledgment的错误
```
# rabbitmqadmin --username=admin --password=V2SG@xdr list queues name messages messages_ready messages_unacknowledged
+----------------------------------------------+----------+----------------+-------------------------+
|                     name                     | messages | messages_ready | messages_unacknowledged |
+----------------------------------------------+----------+----------------+-------------------------+
| hello                                        | 3        | 3              | 0                       |
+----------------------------------------------+----------+----------------+-------------------------+
```

### 消息持久性
以上案例中, 如果RabbitMQ服务器停止，消息会丢失。
```
systemctl restart rabbitmq-server.service

# rabbitmqadmin --username=XXX --password=YYY list queues name messages messages_ready messages_unacknowledged
(重启后队列消失了，消息也丢失了)
+----------------------------------------------+----------+----------------+-------------------------+
|                     name                     | messages | messages_ready | messages_unacknowledged |
+----------------------------------------------+----------+----------------+-------------------------+
+----------------------------------------------+----------+----------------+-------------------------+

```

如果我们希望队列在RabbitMQ节点重启后仍然存在，需要同时在生产者和消费者端将队列声明为持久(durable)的，
```
channel.queue_declare(queue='hello', durable=True)
```
进一步, 如果我们希望消息也是持久的, 需要在basic.publish方法中设置delivery_mode参数为pika.DeliveryMode.Persistent
```
channel.basic_publish(exchange='',
                      routing_key="task_queue",
                      body=message,
                      properties=pika.BasicProperties(
                         delivery_mode = pika.DeliveryMode.Persistent
                      ))
```
注：将消息标记为持久消息不能完全保证消息不丢失。 RabbitMQ不会对每条消息执行fsync(2)——它可能只是保存到缓存中，而不是真正写入磁盘。

### 公平调度
例如有两个worker，所有奇数消息复杂, 处理很慢，而偶数消息很快处理完了。Round-Robin策略仍然会均匀分发消息，对worker负载一无所知。

使用basic_qos方法, 告诉RabbitMQ不要一次给某一个worker发太多的消息。 某个worker正在处理前一条消息, 没有发送确认的情况下，不要给它发的消息 
```
channel.basic_qos(prefetch_count=1)
```

完整的代码如下:
new_task.py
```
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


