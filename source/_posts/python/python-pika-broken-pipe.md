---
title: python pika broken pipe
date: 2017-08-19 18:46:21
tags: python
---


## 问题描述

在消费rabbitMQ队列时, 每次进入回调函数内需要进行一些比较耗时的操作;操作完成后给rabbitMQ server发送ack信号以dequeue本条消息。<br>
问题就发生在发送ack操作时, 程序提示链接已被断开或socket error。
<!--more-->

## 源码示例

```python
#!/usr/bin
#coding: utf-8

import pika
import time


USER = 'guest'
PWD = 'guest'
TEST_QUEUE = 'just4test'

def callback(ch, method, properties, body):
    print(body)
    time.sleep(600)
    ch.basic_publish('', routing_key=TEST_QUEUE, body="fortest")
    ch.basic_ack(delivery_tag = method.delivery_tag)

def test_main():
    s_conn = pika.BlockingConnection(
        pika.ConnectionParameters('127.0.0.1', 
            credentials=pika.PlainCredentials(USER, PWD)))
    chan = s_conn.channel()
    chan.queue_declare(queue=TEST_QUEUE)

    chan.basic_publish('', routing_key=TEST_QUEUE, body="fortest")
    chan.basic_consume(callback, queue=TEST_QUEUE)
    chan.start_consuming()

if __name__ == "__main__":
    test_main()
```

运行一段时间后， 就会报错:

```
[ERROR][pika.adapters.base_connection][2017-08-18 12:33:49]Error event 25, None
[CRITICAL][pika.adapters.base_connection][2017-08-18 12:33:49]Tried to handle an error where no error existed
[ERROR][pika.adapters.base_connection][2017-08-18 12:33:49]Fatal Socket Error: BrokenPipeError(32, 'Broken pipe')
```

## 问题排查

### 猜测：pika客户端没有及时发送心跳，连接被server断开

一开始修改了heartbeat_interval参数值, 示例如下:

```python
def test_main():
    s_conn = pika.BlockingConnection(
        pika.ConnectionParameters('127.0.0.1', 
            heartbeat_interval=10,
            socket_timeout=5,
            credentials=pika.PlainCredentials(USER, PWD)))
    # ....
```
修改后运行依然报错，后来想想应该单线程被一直占用，pika无法发送心跳；<br>
于是又加了个心跳线程, 示例如下:

```python
#!/usr/bin
#coding: utf-8

import pika
import time
import logging
import threading

USER = 'guest'
PWD = 'guest'
TEST_QUEUE = 'just4test'

class Heartbeat(threading.Thread):
    def __init__(self, connection):
        super(Heartbeat, self).__init__()
        self.lock = threading.Lock()
        self.connection = connection
        self.quitflag = False
        self.stopflag = True
        self.setDaemon(True)

    def run(self):
        while not self.quitflag:
            time.sleep(10)
            self.lock.acquire()
            if self.stopflag :
                self.lock.release()
                continue
            try:
                self.connection.process_data_events()
            except Exception as ex:
                logging.warn("Error format: %s"%(str(ex)))
                self.lock.release()
                return
            self.lock.release()

    def startHeartbeat(self):
        self.lock.acquire()
        if self.quitflag==True:
            self.lock.release()
            return
        self.stopflag=False
        self.lock.release()

def callback(ch, method, properties, body):
    logging.info("recv_body:%s" % body)
    time.sleep(600)
    ch.basic_ack(delivery_tag = method.delivery_tag)

def test_main():
    s_conn = pika.BlockingConnection(
        pika.ConnectionParameters('127.0.0.1', 
            heartbeat_interval=10,
            socket_timeout=5,
            credentials=pika.PlainCredentials(USER, PWD)))
    chan = s_conn.channel()
    chan.queue_declare(queue=TEST_QUEUE)
    chan.basic_consume(callback,
                       queue=TEST_QUEUE)

    heartbeat = Heartbeat(s_conn)
    heartbeat.start()          #开启心跳线程
    heartbeat.startHeartbeat()
    chan.start_consuming()

if __name__ == "__main__":
    test_main()
```
尝试运行，结果还是不行，不得不安静下来思考自己是不是想错了。<br>
去看它的api，看到heartbeat_interval的解析:

```
:param int heartbeat_interval: How often to send heartbeats.
                                  Min between this value and server's proposal
                                  will be used. Use 0 to deactivate heartbeats
                                  and None to accept server's proposal.
```
                           
按这样说法，应该还是没有把心跳值给设置好。上面的程序期望是10秒发一次心跳，但是理论上发送心跳的间隔会比10秒多一点。所以艾玛，我应该是把*heartbeat_interval*的作用搞错了， 它是指超过这个时间间隔不发心跳或不给server任何信息，server就会断开连接, 而不是说pika会按这个间隔来发心跳。 结果我把heartbeat_interval值设置高一点(比实际发送心跳/信息的间隔更长)，比如上面设置成60秒，就正常运行了。

如果不指定*heartbeat_interval*， 它默认为None， 意味着按rabbitMQ server的配置来检测心跳是否正常。<br>
如果设置*heartbeat_interval=0*， 意味着不检测心跳，server端将不会主动断开连接。
