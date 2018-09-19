---
title: Centos7安装ElasticSearch6.4
date: 2018-09-19 16:05:51
tags: elasticsearch
---

## centos安装ElasticSearch6.4

### 节点准备

节点IP      | 角色      | ES节点名称
------------|-----------|-----------
192.168.1.10       |master        |node1
192.168.1.11       |data        |node2

<!--more-->

### 1.下载ES安装包

```shell
cd /opt
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.4.0.tar.gz
tar -zxvf elasticsearch-6.4.0.tar.gz
```

### 2. 添加普通用户

```shell
groupadd elsearch
useradd elsearch -g elsearch -p elasticsearch
chown -R elsearch.elsearch /opt/elasticsearch-6.4.0/
```

### 3. 修改ES配置(两个节点分别配置)

```shell
# vim config/elasticsearch.yml

cluster.name: es

## 节点node1
node.name: node1
node.master: true
network.host: 192.168.1.10

## 节点node2
#node.name: node2
#node.master: false
#network.host: 192.168.1.11

node.data: true
path.data: /opt/elasticsearch-6.4.0/data
path.logs: /opt/elasticsearch-6.4.0/logs
transport.tcp.port: 9300
http.port: 9200
discovery.zen.ping.unicast.hosts: ["192.168.1.10:9300","192.168.1.11:9300"]
```

### 修改系统参数

```bash
# vim /etc/security/limits.conf 
elsearch hard nofile 655360
elsearch soft nofile 655360

# vim /etc/sysctl.conf
vm.max_map_count=655360
# sysctl -p
```

### 添加ES到Supervisor

```
# vim /etc/supervisor.d/es.ini
[supervisord]
minfds=65536
minprocs=32768

[program:es-node]
command     = /opt/elasticsearch-6.4.0/bin/elasticsearch
directory   = /opt/elasticsearch-6.4.0
user        = elsearch
startsecs   = 3
redirect_stderr         = true
stdout_logfile_maxbytes = 50MB
stdout_logfile_backups  = 10
stdout_logfile          = /opt/elasticsearch-6.4.0/logs/supervisor.log
```

### 通过Supervisor启动ES服务

```shell
# 两个节点分别启动
supervisorctl reread
supervisorctl add es-node
# supervisorctl start es-node
supervisorctl status
```

### 浏览器测试访问

![es-9200.png](/img/es-9200.png)

### 下载cerebro管理资源包(假设安装在192.168.1.10)

```shell
cd /opt
wget https://github.com/lmenezes/cerebro/releases/download/v0.8.1/cerebro-0.8.1.tgz
tar -zxf cerebro-0.8.1.tgz
chown -R elsearch.elsearch /opt/cerebro-0.8.1
```

### 配置cerebro

```bash
# vim conf/application.conf
hosts = [
  {
    host = "http://192.168.1.10:9200"
    name = "Default ES Cluster"
#    auth = {
#        username = "xx_admin"
#        password = "xx_pwd"
#    }
  },
 }
```
 
 ### 添加cerebro到Supervisor
 
```
 [program:cerebro-node]
command     = /opt/cerebro-0.8.1/bin/cerebro -Dhttp.port=12345 -Dhttp.address=192.168.1.10
directory   = /opt/cerebro-0.8.1/
user        = elsearch
startsecs   = 3

redirect_stderr         = true
stdout_logfile_maxbytes = 100MB
stdout_logfile_backups  = 10
stdout_logfile          = /opt/cerebro-0.8.1/logs/supervisor.log
```

### 通过Supervisor启动cerebro

```
supervisorctl reread
supervisorctl add cerebro-node
# supervisorctl start cerebro-node
supervisorctl status
```

### 浏览器访问cerebro

![es-12345.png](/img/es-12345.png)
