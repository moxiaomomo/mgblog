---
layout: consul
title: 基于Docker的Consul(官方镜像)集群部署指南
date: 2017-07-31 11:47:23
tags:
---


### 关于Consul

Consul 是一个支持多数据中心分布式高可用的服务发现和配置共享的服务软件.<br>
Consul 由 HashiCorp公司用Go语言开发, 基于Mozilla Public License 2.0的协议进行开源. <br>
Consul 支持健康检查,并允许 HTTP 和 DNS 协议调用 API 存储键值对.<br>
命令行超级好用的虚拟机管理软件 vgrant 也是 HashiCorp 公司开发的产品.<br>
一致性协议采用 Raft 算法,用来保证服务的高可用. 使用 GOSSIP 协议管理成员和广播消息, 并且支持 ACL 访问控制.<br>

### 关于官方consul镜像

关于官方consul镜像，三点说明:
+ 想对于其他版本镜像，其stars和pull数量是最多的, pull达到10M+;
+ 官方文档可读性相对不够好, 有些绕;
+ 目前官方镜像更新进度基本与consul最新版本保持一致。

### 多机部署consul:0.8.5版本操作参考

##### 部分参数说明
  - ***--net=host*** docker参数, 使得docker容器越过了net namespace的隔离，免去手动指定端口映射的步骤
  - ***-server*** consul支持以server或client的模式运行, server是服务发现模块的核心, client主要用于转发请求
  - ***-advertise*** 将本机私有IP传递到consul
  - ***-bootstrap-expect*** 指定consul将等待几个节点连通，成为一个完整的集群
  - ***-retry-join*** 指定要加入的consul节点地址，失败会重试, 可多次指定不同的地址
  - ***-client*** consul绑定在哪个client地址上，这个地址提供HTTP、DNS、RPC等服务，默认是127.0.0.1
  - ***-bind*** 该地址用来在集群内部的通讯，集群内的所有节点到地址都必须是可达的，默认是0.0.0.0
  - ***allow_stale*** 设置为true, 表明可以从consul集群的任一server节点获取dns信息, false则表明每次请求都会经过consul server leader


##### 集群的多机部署参考

  * 多中心部署结构图
  
  <img src="/img/consul_archi.png" width="700" height="700" alt="consul部署模式" align=center/><br>
  生产环境下, 一般一个宿主host一个consul节点；<br>
  server节点建议一个数据中心部署3-5个, client节点可部署任意节点。

  * 启动consul server节点, 在对应的机器上root权限下执行以下脚本(需修改节点列表: nodes=(...) ):

```bash
#!/bin/bash

conf_dir=/opt/consul/conf
mkdir -p ${conf_dir}

nodes=(
10.220.16.133
10.220.16.134
10.220.16.163
)
consul_ver=0.8.5
retry_interval=15s
contain_svr_name=consul_server

privIP=$(/sbin/ifconfig eth0 | sed -n 's/.*inet \(addr:\)\?\([0-9.]\{7,15\}\) .*/\2/p')
if [[ !("${nodes[@]}" =~ $privIP) ]];
then
    echo -e "Current node:${privIP} not in configured server nodes.\n"
    exit
fi

svr_runing=$(docker ps -a | grep "${contain_svr_name}" | egrep "Up [About]|[0-9]{1,}")
if [[ ${svr_runing} != "" ]];
then
    echo -e "Current container of consul server has been running.\n"
    exit
else
    svr_exists=$(docker ps -a | grep "${contain_svr_name}")
    if [[ ${svr_exists} != "" ]];
    then
        echo -e "Now try to start the container as it stopped...\n"
        docker start ${svr_exists}
        sleep 2
        docker ps -a grep "${contain_svr_name}"
        exit
    fi
fi

echo -e "To start a new container for consul...\n"
echo -e "To initialize configuration...\n"

nodels=""
for host in ${nodes[*]}
do
    if [[ $nodels != "" ]];
    then
        nodels=$nodels,
    fi
    nodels=$nodels"\"$host\""
done

config="{\n
\"datacenter\": \"dc_dl\",\n
\"retry_join\": [${nodels}],\n
\"retry_interval\": \"${retry_interval}\",\n
\"rejoin_after_leave\": true,\n
\"start_join\": [${nodels}],\n
\"bootstrap_expect\": 1,\n
\"server\": true,\n
\"ui\": true,\n
\"dns_config\": {\"allow_stale\": true, \"max_stale\": \"5s\"},\n
\"node_name\": \"$HOSTNAME\"\n
}\n"

echo $config
echo -e ${config} > ${conf_dir}/server.json
echo -e ${config}

docker run -d -v ${conf_dir}:${conf_dir} \
    --name ${contain_svr_name} \
    --net=host consul:${consul_ver} agent \
    -config-dir=${conf_dir} \
    -client=0.0.0.0 \
    -bind=${privIP} \
    -advertise=${privIP}

sleep 2

svr_runing=$(docker ps -a | grep "${contain_svr_name}" | egrep "Up [About]|[0-9]{1,}")
if [[ ${svr_runing} == "" ]];
then
    echo -e "\nError: docker-consul failed to start...\n"
    exit
fi
echo -e "\nOK: docker-consul has started as background server.\n"
```

  * 启动consul client节点(任意数量或不部署), 在需要部署的机器上执行(需修改svrnodes列表):

```bash
#!/bin/bash

conf_dir=/opt/consul/conf
mkdir -p ${conf_dir}

svrnodes=(
10.220.16.133
10.220.16.134
10.220.16.163
)
privIP=$(/sbin/ifconfig eth0 | sed -n 's/.*inet \(addr:\)\?\([0-9.]\{7,15\}\) .*/\2/p')
consul_ver=0.8.5
retry_interval=15s
contain_cli_name=consul_client

if [[ "${svrnodes[@]}" =~ $privIP ]];
then
    echo -e "Current node:${privIP} is configured for server consul, not to run client mode.\n"
    exit
fi

svr_runing=$(docker ps -a | grep "${contain_cli_name}" | egrep "Up [About]|[0-9]{1,}")
if [[ ${svr_runing} != "" ]];
then
    echo -e "Current container of consul client has been running.\n"
    exit
else
    svr_exists=$(docker ps -a | grep "${contain_cli_name}")
    if [[ ${svr_exists} != "" ]];
    then
        echo -e "Now try to start the container as it stopped...\n"
        docker start ${svr_exists}
        sleep 2
        docker ps -a grep "${contain_cli_name}"
        exit
    fi
fi

echo -e "To start a new container for consul...\n"
echo -e "To initialize configuration...\n"

nodels=""
for host in ${svrnodes[*]}
do
    if [[ $nodels != "" ]];
    then
        nodels=$nodels,
    fi
    nodels=$nodels"\"$host\""
done

config="{\n
\"retry_join\": [${nodels}],\n
\"retry_interval\": \"${retry_interval}\",\n
\"rejoin_after_leave\": true,\n
\"start_join\": [${nodels}],\n
\"server\": false,\n
\"ui\": true,\n
\"node_name\": \"$HOSTNAME\"\n
}\n"

echo $config
echo -e ${config} > ${conf_dir}/client.json
echo -e ${config}

docker run -d -v ${conf_dir}:${conf_dir} \
    --name ${contain_cli_name} \
    --net=host consul:${consul_ver} agent \
    -config-dir=${conf_dir} \
    -client=0.0.0.0 \
    -advertise=${privIP}

sleep 2

svr_runing=$(docker ps -a | grep "${contain_cli_name}" | egrep "Up [About]|[0-9]{1,}")
if [[ ${svr_runing} == "" ]];
then
    echo -e "\nError: docker-consul client node failed to start...\n"
    exit
fi
echo -e "\nOK: docker-consul has started as a client node.\n"
```

  * 检查集群的状态

    通过webUI地址: ***http://<其中一个节点ip>:8500/ui/***可查看集群状态，正常情况下会显示***3 passing***。<br>
    通过docker命令行:<br>

```bash
root@hadoop:~# docker exec -it consul_server consul members
Node                          Address             Status  Type    Build  Protocol  DC
hadoop.slave01.fs.115cdn.net  10.220.16.133:8301  alive   server  0.8.5  2         dc1
hadoop.slave02.fs.115cdn.net  10.220.16.134:8301  alive   server  0.8.5  2         dc1
hadoop.slave03.fs.115cdn.net  10.220.16.163:8301  alive   server  0.8.5  2         dc1
hadoop.slave04.fs.115cdn.net  10.220.16.125:8301  alive   client  0.8.5  2         dc1
```
