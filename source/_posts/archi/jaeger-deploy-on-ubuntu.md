---
title: '[分布式trace]在Ubuntu17.10上部署jaeger'
date: 2018-06-18 16:56:24
tags: ["tracing","docker"]
---

## 关于Jaeger

Jaeger是由Uber发布的一种分布式调用链跟踪系统，主要用于集成到微服务调用追踪，和Zipkin作用类似。通过调用链跟踪系统，可以快速了解各节点的响应状况，方便定位问题。

## Jaeger关键组件

Jaeger本身是一种可单独部署运行的服务，主要有以下几个关键组件：

- 数据存储Cassandra
- 数据收集jaeger-collector
- 客户端代理jaeger-agent
- 客户端库jaeger-client
- 数据库代理jaeger-query
- UI查询jaeger-ui

<!--more-->

## 部署cassandra

首先通过`docker pull`的方式将cassandra镜像拉取下来：

```shell
$docker pull docker.io/cassandra
```

假设这里要部署的cassandra集群为两个节点，利用docker-compose来进行服务编排。
接着是创建一个`docker-compose.yml`文件，并设置好编排内容：

```yml
version: '2'
services:

###############################
   cassandra0:
    image: cassandra
    container_name: cassandra0
    ports:
     - 9042:9042
     - 7199:7199
    mem_limit: 1024M

###############################
   cassandra1:
    image: cassandra
    container_name: cassandra1
    ports:
     - 9142:9042
    links:
     - cassandra0:seed
    environment:
     - CASSANDRA_SEEDS=seed
    mem_limit: 1024M
```

yml文件设置好后在docker中启动canssandra的两个节点：

```shell
$docker-compose up -d
Creating cassandra0 ...
Creating cassandra0 done
Creating cassandra1 ...
Creating cassandra1 done
```

容器节点起来后，检查一下cassandra的运行状态：

```shell
$docker exec cassandra0 nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns (effective)  Host ID                               Rack
UN  172.18.0.2  204.5 KiB  256          100.0%            d7363c7e-6365-4f43-9f0d-cb75797954f7  rack1
UN  172.18.0.3  226.09 KiB  256          100.0%            0ef2f23d-8dbf-4bf0-a73e-f3cf0b65ff1b  rack1
```
可以看到两个节点的状态都是`UN`, UP-Normal, 运行正常。

## 在Cassandra中创建Jaeger相关表

这一步主要是先找到jaegertracing项目中用来初始化表的v001.cql.templ及create.sh脚本，
然后通过cqlsh命令来执行相关初始化操作：

```shell
$pip install cqlsh
$cd $GOPATH/src/github.com/jaegertracing/jaeger/plugin/storage/cassandra/schema/
$DATACENTER=dc1 REPLICATION_FACTOR=1 MODE=prod ./create.sh v001.cql.templ | cqlsh --cqlversion=3.4.4
Using template file v001.cql.templ with parameters:
    mode = prod
    datacenter = dc1
    keyspace = jaeger_v1_dc1
    replication = {'class': 'NetworkTopologyStrategy', 'dc1': '1' }
    trace_ttl = 172800
    dependencies_ttl = 0
```

## 部署Jaeger

有了存储服务后，最后一步我们来部署Jaeger相关的服务组件。

### 1. 下载 [Jaeger安装包](https://github.com/jaegertracing/jaeger/releases/download/v1.5.0/jaeger-1.5.0-linux-amd64.tar.gz)

```bash
$tar -zxvf jaeger-1.5.0-linux-amd64.tar.gz -C /your_path/
$cd /your_path/
$mv jaeger-1.5.0-linux-amd64 jaeger
$cd jaeger
```

### 2. 部署Collector,Agent,Query

```bash
$mkdir log
$nohup ./jaeger-collector --cassandra.keyspace=jaeger_v1_dc1  --cassandra.servers=127.0.0.1 --collector.zipkin.http-port=9411 >> log/collector.log 2>&1 &
$nohup ./jaeger-agent  --collector.host-port=127.0.0.1:14267 >> log/agent.log 2>&1 &
$nohup ./jaeger-query --cassandra.keyspace jaeger_v1_dc1  --cassandra.servers 127.0.0.1 --query.static-files=./jaeger-ui-build/build >> log/query.log 2>&1 &
```

### 3. 打开UI查询页

所有组件都运行起来后，可以这样来访问查询页面， 如打开`http://yourIP:16686/`即可看到主界面：
![JaegerUI](https://img-blog.csdn.net/20180618132747474?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L21veGlhb21vbW8=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
