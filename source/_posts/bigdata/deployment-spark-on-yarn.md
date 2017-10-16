---
title: '[deployment]spark on yarn'
date: 2016-12-16 23:10:04
tags: bigdata
---

### 安装包准备
- Oracle JDK
安装了elasticSearch的系统应该已经配置好了JDK环境; 推荐JDK7
- scala开发环境
spark依赖于scala运行, scala是开发spark统计程序的官方语言; 当前使用scala-2.11版本
- hadoop集群
hadoop-yarn为spark运算提供资源管理及hdfs存储; 当前使用apache hadoop-2.7.3版本
- spark集群
用于分布式运算; 当前使用apache spark 2.0.2版本
- ES-Hadoop插件
es-hadoop作为hadoop/spark集成elasticSearch的插件使用; 当前使用es-hadoop_5.1.1版本

### 安装步骤

** 尚未找到docker对hadoop多机器多节点集群的快速部署方案, 暂时先手动部署

<!--more-->

#### 1. 安装JDK1.7

```bash
mkdir /usr/local/java && cd /usr/local/java
wget "http://download.oracle.com/otn/java/jdk/7u76-b13/jdk-7u76-linux-x64.tar.gz"
tar -zxf jdk-7u76-linux-x64.tar.gz && rm -f jdk-7u76-linux-x64.tar.gz
```

在/etc/profile中加入如下变量:

```bash
export JAVA_HOME=/usr/local/java/jdk1.7.0_76
export JRE_HOME=/usr/local/java/jdk1.7.0_76/jre
export PATH=$PATH:/usr/local/java/jdk1.7.0_76/bin
export CLASSPATH=./:/usr/local/java/jdk1.7.0_76/lib:/usr/local/java/jdk1.7.0_76/jre/lib
```

让配置生效: *source /etc/profile*

#### 2. 安装scala-2.11<br>
```bash
  mkdir /usr/local/scala && cd /usr/local/scala/
  wget "http://downloads.lightbend.com/scala/2.11.8/scala-2.11.8.tgz"
  tar -zxf scala-2.11.8.tgz && rm -f scala-2.11.8.tgz
  echo "export SCALA_HOME=/usr/local/scala/scala-2.11.8" >> /etc/profile
  source /etc/profile
  echo "export PATH=$SCALA_HOME/bin:$PATH" >> /etc/profile
  source /etc/profile
```

#### 3. 部署Hadoop

##### 为hadoop创建专有用户

```bash
    groupadd hadoop           # 添加hadoop用户组
    useradd hadoop -g hadoop  # 添加hadoop用户并加入hadoop组
```

vim /etc/sudoers          # 编辑sudoers文件，给hadoop用户sudo权限
    
``` 
hadoop  ALL=(ALL) ALL    # 在sudoers末尾加上这一行
```

##### 修改各机器主机名, 用于方便区分节点
假设有三台机器, 一个用作master节点, 两个用于slave节点，如下:
> 192.168.1.181 master
> 192.168.1.191 slave01
> 192.168.1.102 slave02
    
那么在将各个hostname分别改为master, slave01, slave02后, 各自配置/etc/hosts:<br>

```bash
    echo "192.168.1.181 master" >> /etc/hosts
    echo "192.168.1.191 slave01" >> /etc/hosts
    echo "192.168.1.102 slave02" >> /etc/hosts
```

##### 配置免密码登录
hadoop集群中需要配置namenode(master节点)通过用户hadoop免密码登录到本地以及其他datanode(slave节点);<br>
具体做法是将master节点上的rsa这类证书分发到各个slave节点对应ssh配置目录， 这里略过具体过程。

##### 下载hadoop2.7.3<br>
```bash
    mkdir /usr/local/hadoop && cd /usr/local/hadoop
    wget "https://archive.apache.org/dist/hadoop/core/hadoop-2.7.3/hadoop-2.7.3.tar.gz"
    tar -zxf hadoop-2.7.3.tar.gz && rm -f hadoop-2.7.3.tar.gz 
    mkdir -p /usr/local/hadoop/hdfs/data
    mkdir -p /usr/local/hadoop/hdfs/name
    mkdir -p /usr/local/hadoop/tmp
    chown -R hadoop:hadoop /usr/local/hadoop
    cd hadoop-2.7.3 && su hadoop
```

##### 配置环境变量(所有节点同样配置)<br>
```bash
    echo "export HADOOP_HOME=/usr/local/hadoop/hadoop-2.7.3" >> /etc/profile
    echo "export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin" >> /etc/profile
    source /etc/profile
```

##### 修改配置文件<br>
在${HADOOP_HOME}/etc/hadoop/下(***可先在主节点中配置好, 然后拷贝到其他工作节点***)<br>

1) vim etc/hadoop/hadoop-env.sh

```bash
export JAVA_HOME=/usr/local/java/jdk1.7.0_76
```

2) vim etc/hadoop/yarn-env.sh

```bash
export JAVA_HOME=/usr/local/java/jdk1.7.0_76
```

3) vim etc/hadoop/slaves  // 把datanode的hostname写入slaves文件, 根据实际情况修改

```
   slave01
   slave02
```

4) vim etc/hadoop/core-site.xml
  
```xml
<configuration>
  <property>
      <name>fs.defaultFS</name>
      <value>hdfs://master:9000</value>
      <description>HDFS的URI，文件系统://namenode标识:端口号</description>
  </property>          
  <property>
      <name>hadoop.tmp.dir</name>
      <value>/usr/local/hadoop/tmp</value>
      <description>namenode上本地的hadoop临时文件夹</description>
  </property>
</configuration>
```

5) vim etc/hadoop/hdfs-site.xml

```xml
      <configuration>
          <property>
              <name>dfs.name.dir</name>
              <value>file:/usr/local/hadoop/hdfs/name</value>
              <description>namenode上存储hdfs名字空间元数据 </description> 
          </property>
          
          <property>
              <name>dfs.data.dir</name>
              <value>file:/usr/local/hadoop/hdfs/data</value>
              <description>datanode上数据块的物理存储位置</description>
          </property>
          
          <property>
              <name>dfs.replication</name>
              <value>2</value>
              <description>副本个数，配置默认是3,应小于datanode机器数量</description>
          </property>
          <property>
              <name>dfs.client.read.shortcircuit</name>
              <value>true</value>
          </property>
          <property>
              <name>dfs.domain.socket.path</name>
              <value>/var/lib/hadoop-hdfs/dn_socket</value>
          </property>
      </configuration>
```

6) vim etc/hadoop/yarn-site.xml

```xml
      <configuration>
          <property>
              <name>yarn.nodemanager.aux-services</name>
              <value>mapreduce_shuffle</value>
          </property>
          <property>
              <name>yarn.resourcemanager.hostname</name>
              <value>master</value>
          </property>
      </configuration>
```

7) vim etc/hadoop/mapred-site.xml
```xml
      <configuration>
          <property>
              <name>mapreduce.framework.name</name>
              <value>yarn</value>
          </property>
      </configuration>
```

  *所有配置文件修改后, 将/usr/local/hadoop/文件夹拷贝到datanode中相应的位置*

##### hadoop集群初始化及启动, 在主节点中执行
```bash
cd /usr/local/hadoop/hadoop-2.7.3 && su hadoop
bin/hdfs namenode -format
sbin/start-dfs.sh
sbin/start-yarn.sh
```
***hadoop启动后, 通过http://master:50070/和http://master:8088/可以分别查看hdfs和task等状态信息***
同时也可以用jps查看:
```
# 主节点namenode
hadoop@1:/usr/local/hadoop$ jps
9378 NameNode
10218 NodeManager
9786 SecondaryNameNode
10077 ResourceManager
11453 Jps
9550 DataNode

# 工作节点
hadoop@2:/usr/local/hadoop$ jps
1489 Jps
796 NodeManager
606 DataNode
```

#### 4. 部署spark
##### 下载spark-2.0.2-bin-hadoop2.7.tgz
```bash
mkdir /usr/local/spark/ && cd /usr/local/spark
chown -R hadoop:hadoop /usr/local/spark
wget "http://d3kbcqa49mib13.cloudfront.net/spark-2.0.2-bin-hadoop2.7.tgz"
tar -zxf spark-2.0.2-bin-hadoop2.7.tgz && rm -f spark-2.0.2-bin-hadoop2.7.tgz
cd spark-2.0.2-bin-hadoop2.7
```

##### 配置环境变量 
```bash
echo "export SPARK_HOME=/usr/local/spark/spark-2.0.2-bin-hadoop2.7" >> /etc/profile
source /etc/profile
echo "export PATH=$PATH:$SPARK_HOME/bin:$SPARK_HOME/sbin" >> /etc/profile
source /etc/profile
```

##### 修改配置文件

+ vim conf/spark-env.sh
```bash
# export SPARK_SSH_OPTS="-p 23456"
export JAVA_HOME=/usr/local/java/jdk1.7.0_76
export SCALA_HOME=/usr/local/scala/scala-2.11.8
export SPARK_HOME=/usr/local/spark/spark-2.0.2-bin-hadoop2.7
export HADOOP_HOME=/usr/local/hadoop/hadoop-2.7.3
export SPARK_MASTER_HOST=master
export HADOOP_CONF_DIR=/usr/local/hadoop/hadoop-2.7.3/etc/hadoop/
export SPARK_HISTORY_OPTS="-Dspark.history.retainedApplications=3 -Dspark.history.fs.logDirectory=hdfs://master:9000/sparklogs"
export LD_LIBRARY_PATH=${HADOOP_HOME}/lib/native/:$LD_LIBRARY_PATH
```
+ vim spark-default.conf

```
spark.yarn.jars hdfs://1.downlog-es.10.220.2.233.fs.115cdn.net:9100/sparkjars
#spark.yarn.archive hdfs://hadoop.master.fs.115cdn.net:9000/user/btadmin/.sparkStaging/application_1483167145424_0004/__spark_libs__8967711398799981387.zip
spark.eventLog.enabled  true
spark.eventLog.dir hdfs://1.downlog-es.10.220.2.233.fs.115cdn.net:9100/sparklogs
spark.history.fs.logDirectory hdfs://1.downlog-es.10.220.2.233.fs.115cdn.net:9100/sparkHistoryLogs

#spark.executor.extraClassPath /usr/local/spark/jars/mysql-connector-java-5.1.40.jar
#spark.executor.extraClassPath /usr/local/spark/extrajars/mysql-connector-java-5.1.40.jar:/usr/local/spark/extrajars/mongo-spark-connector_2.11-0.2.jar
#spark.mongodb.input.uri mongodb://10.220.2.223:27017/hashmap.fidsha1
spark.executor.instances  12
spark.driver.cores        2
spark.executor.cores      2
spark.executor.memory     4g
spark.driver.memory       4g
spark.default.parallelism 48
spark.sql.crossJoin.enabled true
spark.serializer org.apache.spark.serializer.KryoSerializer
```

+ vim slaves
```
slave01
slave02
```

+ 在hdfs为Spark建立必要的目录

```bash
# SPARK_HOME
# sprk集群的日志目录：配置文件中对history-server中定义的log目录
$ hdfs dfs -mkdir /sparklogs
# 将spark的jar包拷贝到hadoop服务器上，这样避免每次计算的时候都要做去一次拷贝操作
$ hdfs dfs -mkdir /sparkjars
$ cd /usr/local/spark/spark-2.0.2-bin-hadoop2.7/ && hdfs dfs -put jars/* /sparkjars/
```

  ***配置文件修改完成后， 将/usr/local/spark文件夹拷贝到其他节点对应的位置, 并配置好环境变量***

+ spark集群启动, 在主节点中执行:
```bash
cd /usr/local/spark/spark-2.0.2-bin-hadoop2.7/ && ./sbin/start-all.sh
```
用自带example验证测试
```bash
root@master:/usr/local/spark/spark-2.0.2-bin-hadoop2.7# bin/spark-submit --class org.apache.spark.\
examples.JavaSparkPi --master spark://master:7077 examples/jars/spark-examples_2.11-2.0.2.jar
16/12/26 15:41:13 WARN SparkContext: Use an existing SparkContext, some configuration may not take effect.
[Stage 0:>                                                          (0 + 0) / 2]16/12/26 15:41:20 WARN \
TaskSetManager: Stage 0 contains a task of very large size (981 KB). The maximum recommended task size is 100 KB.
Pi is roughly 3.13608 
```
  *spark启动后， 通过http://master:8080/可以查看spark当前的运行状态*

#### 5. 结合es-hadoop
+ 下载ES-Hadoop

```bash
mkdir /usr/local/es-hadoop && cd /usr/local/es-hadoop
wget "http://download.elastic.co/hadoop/elasticsearch-hadoop-5.1.1.zip"
unzip elasticsearch-hadoop-5.1.1.zip && rm -f elasticsearch-hadoop-5.1.1.zip
cp elasticsearch-hadoop-5.1.1/dist/elasticsearch-hadoop-5.1.1.jar /usr/local/spark/spark-2.0.2-bin-hadoop2.7/jars/
```

+ 通过spark访问/操作elasticSearch
```
root@master:/usr/local/spark/spark-2.0.2-bin-hadoop2.7# ./bin/spark-submit your_spark_es_script.py
```
