---
title: 'ElasticSearch数据备份与恢复'
date: 2017-10-31 11:26:17
tags: elasticsearch 备份恢复
---

本文主要是记录了(Ubuntu环境)通过sshfs共享文件系统来进行快照方式备份数据。
假设ES集群有三个节点:
```
192.168.1.10
192.168.1.11
192.168.1.12
```

### 1. 创建共享目录

- 每个节点安装sshfs, 创建相同目录用于挂载共享目录
```
apt-get install fuse sshfs
mkdir /mnt/backup
```

- 选取其中一个节点的目录(非系统盘)作为共享目录, 设置其他节点免密码登录到该节点
假设选择的节点ip为192.168.1.10:
<!--more-->
```
# 192.168.1.10上创建目录
mkdir /data/es_backup
# 每个节点挂载同样操作挂载/mnt/backup
sshfs 192.168.1.10:/data/es_backup /mnt/backup -o allow_other
# 如果修改了默认ssh端口, 比如23566, 则可以这样:
# sshfs 192.168.1.10:/data/es_backup /mnt/backup -p 23566 -o allow_other
```
其中的参数`-o allow_other`允许了其他用户访问这个目录。

- 测试运行ES的用户对共享目录是否有写权限
```
sudo -u elasticsearch touch /mnt/backup/test
```

### 2. 修改ES配置

- 在elasticsearch.yml中加入一行:
```
path.repo:  /mnt/backup/
```
- 或者直接修改/etc/init.d/elasticsearch, 

将原来的`DAEMON_OPTS`选项
```
DAEMON_OPTS="-d -p $PID_FILE --default.path.home=$ES_HOME --default.path.logs=$LOG_DIR --default.path.data=$DATA_DIR --default.path.conf=$CONF_DIR"
```
修改为:
```
REPO_DIR=/mnt/backup
DAEMON_OPTS="-d -p $PID_FILE --default.path.home=$ES_HOME --default.path.logs=$LOG_DIR --default.path.data=$DATA_DIR --default.path.conf=$CONF_DIR --default.path.repo=$REPO_DIR"
```
具体可能遇到的问题可参考这里: https://discuss.elastic.co/t/path-repo-is-empty/25166/4<br>
修改后重启es集群。

### 3. 创建备份仓库
```
curl -XPUT 'http://192.168.1.10:9200/_snapshot/EsBackup_zip' -d '{
"type": "fs",
"settings": {
    "location": "/mnt/backup/compress_snapshot",
    "compress": true
    }
}'
```
成功后结果返回`{"acknowledged":true}`. 这时查看刚创建的仓库:
```
curl -XGET 'http://192.168.1.10:9200/_snapshot?pretty'
```
正常结果返回:
```
{
  "EsBackup_zip" : {
    "type" : "fs",
    "settings" : {
      "compress" : "true",
      "location" : "/mnt/backup/compress_snapshot"
    }
  }
}
```

### 4. 备份指定索引数据
假设要备份单个索引, 索引名为: user_behavior_201702
```
curl -XPUT 'http://192.168.1.10:9200/_snapshot/EsBackup_zip/snapshot_user_behavior_201702' -d '{"indices": "user_behavior_201702"}'
```
提交备份快照请求后, 查看备份状态:
```
curl -XGET 'http://192.168.1.12:9200/_snapshot/EsBackup_zip/snapshot_user_behavior_201702?pretty'
```
-----
假设要备份多个索引, 比如idx_1, idx_2, idx_3, 则可以:
```
curl -XPUT 'http://192.168.1.10:9200/_snapshot/EsBackup_zip/snapshot_some_name' -d '{"indices": "idx_1,idx_2,idx_3"}'
```
假设要备份全部索引数据, 则可以:
```
curl -XPUT 'http://192.168.1.10:9200/_snapshot/EsBackup_zip/snapshot_all'
```

### 5. 恢复备份索引

删除已备份的索引:
```
curl -XDELETE "http://192.168.1.10:9200/user_behavior_201702"
```
恢复单个索引:
```
curl -XPOST 'http://192.168.1.10:9200/_snapshot/EsBackup_zip/snapshot_user_behavior_201702/_restore' -d '{
    "indices": "user_behavior_201702", 
    "rename_replacement": "restored_ub_201702"
}'
```
恢复整个快照索引:
```
curl -XPOST 'http://192.168.1.10:9200/_snapshot/EsBackup_zip/snapshot_some_name/_restore'
```
提交请求成功后返回`{"accepted":true}`。<br>
查看恢复状态:
```
curl -XGET "http://192.168.1.10:9200/_snapshot/EsBackup_zip/snapshot_user_behavior_201702/_status"
```
