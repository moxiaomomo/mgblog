---
title: flume-kafka部署总结
date: 2017-10-16 18:16:34
tags: flume-kafka
---


### 部署准备

配置日志收集系统(flume+kafka), 版本:
```
apache-flume-1.8.0-bin.tar.gz
kafka_2.11-0.10.2.0.tgz
```
假设部署在三个工作节点上:
```
192.168.0.2
192.168.0.3
192.168.0.4
```

### flume配置说明

假设flume的工作目录在/usr/local/flume, 
监测某日志文件(如/tmp/testflume/chklogs/chk.log),
则新增配置: /usr/local/flume/conf/flume-kafka.conf
<!--more-->

```
Flume2KafkaAgent.sources=mysource
Flume2KafkaAgent.channels=mychannel
Flume2KafkaAgent.sinks=mysink

Flume2KafkaAgent.sources.mysource.type=exec
Flume2KafkaAgent.sources.mysource.channels=mychannel
Flume2KafkaAgent.sources.mysource.command=tail -F /tmp/testflume/chklogs/chk.log

Flume2KafkaAgent.sinks.mysink.channel=mychannel
Flume2KafkaAgent.sinks.mysink.type=org.apache.flume.sink.kafka.KafkaSink
Flume2KafkaAgent.sinks.mysink.kafka.bootstrap.servers=192.168.0.2:9092,192.168.0.3:9092,192.168.0.4:9092
Flume2KafkaAgent.sinks.mysink.kafka.topic=apilog
Flume2KafkaAgent.sinks.mysink.kafka.batchSize=20
Flume2KafkaAgent.sinks.mysink.kafka.requiredAcks=1

Flume2KafkaAgent.channels.mychannel.type=memory
Flume2KafkaAgent.channels.mychannel.capacity=30000
Flume2KafkaAgent.channels.mychannel.transactionCapacity=100

```
三个节点均执行启动命令:
```shell
hadoop@1:/usr/local/flume$ bin/flume-ng agent -c conf -f conf/flume-kafka.conf -n Flume2KafkaAgent
```

### kafka配置说明

假设kafaka的工作目录在/usr/local/kafka,
修改/usr/local/kafka/config/server.properties以下几项:
```
broker.id=0   # broker id，每个节点必须不同， 比如三个节点分别为0, 1, 2
port=9092
advertised.listeners=PLAINTEXT://your_local_ip:9092 #当前节点ip
zookeeper.connect=192.168.0.2:2181,192.168.0.3:2181,192.168.0.4:2181
```
每个节点启动zookeeper:
```shell
hadoop@1:/usr/local/kafka$ bin/zookeeper-server-start.sh
```
每个节点启动kafka:
```shell
hadoop@1:/usr/local/kafka$ bin/kafka-server-start.sh
```
通过jps查看java进程:
```shell
hadoop@1:/usr/local/kafka$ jps
20225 Jps
13286 Kafka
15421 QuorumPeerMain
```

### 验证测试

- 模拟生成日志, 写入/tmp/testflume/chklogs/chk.log, 如:
```bash
#!/bin/bash

chkdir=/tmp/testflume/chklogs/
mkdir -p $chkdir

for((i=0; i<10000; i++));
do
  dt=$(date '+%Y%m%d %H:%M:%S')
  echo "current datetime: $dt" >> $chkdir/chk.log
  sleep 0.5
done
```
- 启动kafka消费者， 查看数据接收记录:
```shell
hadoop@1:/usr/local/kafka$ bin/kafka-console-consumer.sh --bootstrap-server 192.168.0.2:9092,192.168.0.3:2181,192.168.0.4:9092 --from-beginning --topic apilog
current datetime: 20171012 17:42:05
current datetime: 20171012 17:42:06
current datetime: 20171012 17:42:07
current datetime: 20171012 17:42:08
current datetime: 20171012 17:42:09
current datetime: 20171012 17:42:10
current datetime: 20171012 17:42:11
current datetime: 20171012 17:42:12
current datetime: 20171012 17:42:13
```
由此可见, kafka已可以持续接收到日志数据。
