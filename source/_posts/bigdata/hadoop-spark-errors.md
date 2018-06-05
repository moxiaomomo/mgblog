---
title: hadoop-spark错误问题总结
date: 2017-10-16 14:55:21
tags: bigdata
---

## hadoop-spark错误问题总结

#### 1.Caused by: java.lang.NoClassDefFoundError: scala/collection/GenTraversableOnce$class

具体错误日志:
```
Caused by: java.lang.NoClassDefFoundError: scala/collection/GenTraversableOnce$class
    at org.elasticsearch.spark.rdd.AbstractEsRDDIterator.<init>(AbstractEsRDDIterator.scala:28)
    at org.elasticsearch.spark.sql.ScalaEsRowRDDIterator.<init>(ScalaEsRowRDD.scala:49)
    at org.elasticsearch.spark.sql.ScalaEsRowRDD.compute(ScalaEsRowRDD.scala:45)
    at org.elasticsearch.spark.sql.ScalaEsRowRDD.compute(ScalaEsRowRDD.scala:38)
    at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:319)
    at org.apache.spark.rdd.RDD.iterator(RDD.scala:283)
    at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
    at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:319)
    at org.apache.spark.rdd.RDD.iterator(RDD.scala:283)
    at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
    at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:319)
    at org.apache.spark.rdd.RDD.iterator(RDD.scala:283)
    at org.apache.spark.rdd.MapPartitionsRDD.compute(MapPartitionsRDD.scala:38)
    at org.apache.spark.rdd.RDD.computeOrReadCheckpoint(RDD.scala:319)
    at org.apache.spark.rdd.RDD.iterator(RDD.scala:283)
    at org.apache.spark.scheduler.ResultTask.runTask(ResultTask.scala:70)
    at org.apache.spark.scheduler.Task.run(Task.scala:86)
    at org.apache.spark.executor.Executor$TaskRunner.run(Executor.scala:274)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
    ... 1 more
```
<!--more-->
原因分析及解决方法:

- spark所依赖的scala版本与系统安装的scala版本不一致<br>
    通过检查, 系统安装了scala2.11.1, spark依赖的包为2.11.8; 统一之后， 问题依然存在
- scala2.11.8中并不存在类GenTraversableOnce<br>
    通过其API Reference可知, 该类是有定义的， 而且spark下关于sparkSQL的example运行测试通过;
- 代码自身问题, 调用API方式或配置不当<br>
原有代码(参照了官方scala示例):
```python
conf = SparkConf()
conf.setMaster("spark://192.168.1.181:7077").setAppName("es_spark")
conf.set("es.nodes", "192.168.1.181")
conf.set("es.port", "9200")
conf.set("es.resource", "web-2016.12.26/web")
sc = SparkContext(conf=conf)
sqc = SQLContext(sc)
df = sqc.read.format("org.elasticsearch.spark.sql"  ).load("web-2016.12.26/web")
df.printSchema()
print df.first()   # 这里开始报错
```
改用根据rdd来创建view的方法, 修改后如下:
```python
conf = {
    "es.nodes": "192.168.1.181",
    "es.port": "9200",
    "es.resource": "web-2016.12.26/web",
    "es.read.field.include": "city,uri,unique",
}
query = '''{
      "query": {
        "match": {"city":"Guangzhou"}
      }
  }'''
conf['es.query'] = query
rdd = sc.newAPIHadoopRDD(
    "org.elasticsearch.hadoop.mr.EsInputFormat",
    "org.apache.hadoop.io.NullWritable",
    "org.elasticsearch.hadoop.mr.LinkedMapWritable",
    conf=es_conf).map(lambda x: x[1])
schemaUsers = ss.createDataFrame(rdd)
schemaUsers.createOrReplaceTempView("users")
res = ss.sql("select uri from users")
print (res.first())
```

#### 2. NativeCodeLoader: Unable to load native-hadoop library for your platform

原因分析及解决方法:

>主要是jre目录下缺少了libhadoop.so和libsnappy.so两个文件。具体是，spark-shell依赖的是scala，scala依赖的是JAVA_HOME下的jdk，libhadoop.so和libsnappy.so两个文件应该放到$JAVA_HOME/jre/lib/amd64下面。
>这两个so：libhadoop.so和libsnappy.so。前一个so可以在HADOOP_HOME下找到，如hadoop\lib\native。第二个libsnappy.so需要下载一个snappy-1.1.0.tar.gz，然后./configure，make编译出来，编译成功之后在.libs文件夹下。
>当这两个文件准备好后再次启动spark shell不会出现这个问题.
参考自: https://www.zhihu.com/question/23974067/answer/26267153


#### 3. Initial job has not accepted any resources; check your cluster UI to ensure that workers are registered and have sufficient memory

原因分析及解决方法:<br>
默认情况下spark.cores.max是不限制的， 运行第一个spark application时， 会分配所有的cpu来执行任务；<br>
因此第二个application很可能就得不到足够的资源了。假设每个任务最多获取一半的cpu资源， 假设总共节点的core为8, 我们可以设置为spark.cores.max=4。

#### 4. NoSuchMethodError: org.apache.hadoop.http.HttpConfig.isSecure()Z
具体错误日志:
```
btadmin@master:/usr/local/spark/$ bin/spark-submit --class org.apache.spark.examples.JavaSparkPi --master yarn --deploy-mode client examples/jars/spark-examples_2.11-2.0.2.jar
16/12/30 10:00:06 WARN Utils: Your hostname, node1 resolves to a loopback address: 127.0.1.1; using 192.168.1.191 instead (on interface eth0)
16/12/30 10:00:07 WARN Utils: Set SPARK_LOCAL_IP if you need to bind to another address
Exception in thread "main" java.lang.NoSuchMethodError: org.apache.hadoop.http.HttpConfig.isSecure()Z
 at org.apache.hadoop.yarn.webapp.util.WebAppUtils.getResolvedRMWebAppURLWithoutScheme(WebAppUtils.java:101)
 at org.apache.hadoop.yarn.webapp.util.WebAppUtils.getProxyHostAndPort(WebAppUtils.java:90)
 at org.apache.spark.deploy.yarn.YarnRMClient.getAmIpFilterParams(YarnRMClient.scala:115)
 at org.apache.spark.deploy.yarn.ApplicationMaster.addAmIpFilter(ApplicationMaster.scala:582)
 at org.apache.spark.deploy.yarn.ApplicationMaster.runExecutorLauncher(ApplicationMaster.scala:406)
 at org.apache.spark.deploy.yarn.ApplicationMaster.run(ApplicationMaster.scala:247)
 at org.apache.spark.deploy.yarn.ApplicationMaster$$anonfun$main$1.apply$mcV$sp(ApplicationMaster.scala:749)
 at org.apache.spark.deploy.SparkHadoopUtil$$anon$1.run(SparkHadoopUtil.scala:71)
 at org.apache.spark.deploy.SparkHadoopUtil$$anon$1.run(SparkHadoopUtil.scala:70)
 at java.security.AccessController.doPrivileged(Native Method)
 at javax.security.auth.Subject.doAs(Subject.java:415)
 at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1698)
 at org.apache.spark.deploy.SparkHadoopUtil.runAsSparkUser(SparkHadoopUtil.scala:70)
 at org.apache.spark.deploy.yarn.ApplicationMaster$.main(ApplicationMaster.scala:747)
 at org.apache.spark.deploy.yarn.ExecutorLauncher$.main(ApplicationMaster.scala:774)
 at org.apache.spark.deploy.yarn.ExecutorLauncher.main(ApplicationMaster.scala)
```

原因分析及解决方法:
spark中部分jar文件与hadoop的不兼容, 当前spark-defaults.conf中设置:
```
spark.yarn.jars hdfs:///sparkjars/*   # jar文件从${SPARK_HOME}/jars/中已上传到hdfs
```
使用另一种方法代替jar库引用:
```
spark.yarn.archive hdfs://master:9000/user/btadmin/.sparkStaging/application_1483167145424_0004/__spark_libs__8967711398799981387.zip
```

#### 5.Could not find or load main class org.apache.spark.deploy.yarn.ExecutorLauncher

spark-defaults.conf的参数spark.yarn.jars配置错了， 参考:
http://blog.csdn.net/dax1n/article/details/53001884

需要将以下配置项(假设jar库都放在了hdfs)
> spark.yarn.jars hdfs://your.hdfs.host:port/sparkjars/

修改为
> spark.yarn.jars hdfs://your.hdfs.hos:port/sparkjars/*

#### 6.java.nio.channels.ClosedChannelException

具体错误日志:
```
java.nio.channels.ClosedChannelException
    at io.netty.channel.AbstractChannel$AbstractUnsafe.write(...)(Unknown Source)
```

参考博客: http://www.bijishequ.com/detail/203655?p=58<br>
目测是给节点分配的内存太小, yarn直接kill了task。

解决方案：

修改yarn-site.xml，添加下列property
```
<property>
    <name>yarn.nodemanager.pmem-check-enabled</name>
    <value>false</value>
</property>

<property>
    <name>yarn.nodemanager.vmem-check-enabled</name>
    <value>false</value>
</property>
```

#### 7.提交spark作业时, 一直卡在yarn.Client: Application report for application_xxx (state: ACCEPTED)

具体错误日志:
```
17/10/13 15:14:49 INFO impl.YarnClientImpl: Application submission is not finished, submitted application application_1507620566574_0010 is still in NEW
17/10/13 15:14:51 INFO impl.YarnClientImpl: Application submission is not finished, submitted application application_1507620566574_0010 is still in NEW
17/10/13 15:14:52 INFO impl.YarnClientImpl: Submitted application application_1507620566574_0010
17/10/13 15:14:53 INFO yarn.Client: Application report for application_1507620566574_0010 (state: ACCEPTED)
17/10/13 15:14:53 INFO yarn.Client: 
	 client token: N/A
	 diagnostics: N/A
	 ApplicationMaster host: N/A
	 ApplicationMaster RPC port: -1
	 queue: default
	 start time: 1507878873743
	 final status: UNDEFINED
	 tracking URL: http://1.downlog-es.10.220.2.233.fs.115cdn.net:8088/proxy/application_1507620566574_0010/
	 user: hadoop
17/10/13 15:14:54 INFO yarn.Client: Application report for application_1507620566574_0010 (state: ACCEPTED)
17/10/13 15:14:55 INFO yarn.Client: Application report for application_1507620566574_0010 (state: ACCEPTED)
17/10/13 15:14:56 INFO yarn.Client: Application report for application_1507620566574_0010 (state: ACCEPTED)
17/10/13 15:14:57 INFO yarn.Client: Application report for application_1507620566574_0010 (state: ACCEPTED)
17/10/13 15:14:58 INFO yarn.Client: Application report for application_1507620566574_0010 (state: ACCEPTED)
17/10/13 15:14:59 INFO yarn.Client: Application report for application_1507620566574_0010 (state: ACCEPTED)
17/10/13 15:15:00 INFO yarn.Client: Application report for application_1507620566574_0010 (state: ACCEPTED)
17/10/13 15:15:01 INFO yarn.Client: Application report for application_1507620566574_0010 (state: ACCEPTED)
17/10/13 15:15:02 INFO yarn.Client: Application report for application_1507620566574_0010 (state: ACCEPTED)
17/10/13 15:15:03 INFO yarn.Client: Application report for application_1507620566574_0010 (state: ACCEPTED)
17/10/13 15:15:04 INFO yarn.Client: Application report for application_1507620566574_0010 (state: ACCEPTED)
17/10/13 15:15:05 INFO yarn.Client: Application report for application_1507620566574_0010 (state: ACCEPTED)
17/10/13 15:15:06 INFO yarn.Client: Application report for application_1507620566574_0010 (state: ACCEPTED)
17/10/13 15:15:07 INFO yarn.Client: Application report for application_1507620566574_0010 (state: ACCEPTED)

```

检查查yarn配置时, 发现	yarn.resourcemanager.address 的值不正常。

比如节点hostname为: 1.test.10.220.2.233.mycdn.net, 则yarn中参数值为:
```
<property>
    <name>yarn.resourcemanager.address</name>
    <value>1.1.test.10.220.2.233.mycdn.net:8032</value>
    <source>programatically</source>
</property>
```
同时在执行stop-dfs.sh时, 发现namenode和datanode的hostname对不上:
```
Stopping namenodes on [1.1.test.10.220.2.233.mycdn.net]
1.1.test.10.220.2.233.mycdn.net: stopping namenode
1.test.10.220.2.233.mycdn.net: stopping datanode
2.test.10.220.2.234.mycdn.net: stopping datanode
Stopping secondary namenodes [0.0.0.0]
0.0.0.0: stopping secondarynamenode
```
查看/etc/hosts时， 发现多了一个域名映射:
```
10.220.2.233 1.1.test.10.220.2.233.mycdn.net 1
```
注释掉之后， 重启hadoop集群, spark on yarn任务可以正常执行。

#### 8.java.lang.NoClassDefFoundError: kafka/common/TopicAndPartition
具体错误日志:
```
Exception in thread "Thread-3" java.lang.NoClassDefFoundError: kafka/common/TopicAndPartition
	at java.lang.Class.getDeclaredMethods0(Native Method)
	at java.lang.Class.privateGetDeclaredMethods(Class.java:2701)
	at java.lang.Class.privateGetPublicMethods(Class.java:2902)
	at java.lang.Class.getMethods(Class.java:1615)
	at py4j.reflection.ReflectionEngine.getMethodsByNameAndLength(ReflectionEngine.java:345)
	at py4j.reflection.ReflectionEngine.getMethod(ReflectionEngine.java:305)
	at py4j.reflection.ReflectionEngine.getMethod(ReflectionEngine.java:326)
	at py4j.Gateway.invoke(Gateway.java:272)
	at py4j.commands.AbstractCommand.invokeMethod(AbstractCommand.java:132)
	at py4j.commands.CallCommand.execute(CallCommand.java:79)
	at py4j.GatewayConnection.run(GatewayConnection.java:214)
	at java.lang.Thread.run(Thread.java:745)
Caused by: java.lang.ClassNotFoundException: kafka.common.TopicAndPartition
	at java.net.URLClassLoader.findClass(URLClassLoader.java:381)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:331)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
	... 12 more
ERROR:root:Exception while sending command.
Traceback (most recent call last):
  File "/usr/local/spark/python/lib/py4j-0.10.4-src.zip/py4j/java_gateway.py", line 883, in send_command
    response = connection.send_command(command)
  File "/usr/local/spark/python/lib/py4j-0.10.4-src.zip/py4j/java_gateway.py", line 1040, in send_command
    "Error while receiving", e, proto.ERROR_ON_RECEIVE)
Py4JNetworkError: Error while receiving
```
原因及解决方法:
使用的spark-stream-kafka jar包版本不对, spark2.2(python)具体看这里:
https://spark.apache.org/docs/latest/streaming-kafka-integration.html

同时在maven上面下载正确的版本(spark-streaming-kafka-0-8-assembly_2.11):
http://search.maven.org/#search%7Cga%7C1%7Cspark-streaming-kafka

检查jar是否包含该类:
```
hadoop@1:/data/test$ jar -tvf spark-streaming-kafka-0-8-assembly_2.11-2.2.0.jar | grep -i Topicandpartition
  1959 Fri Jun 30 17:08:54 CST 2017 kafka/common/TopicAndPartition$.class
  6136 Fri Jun 30 17:08:54 CST 2017 kafka/common/TopicAndPartition.class
hadoop@1:/data/test$ 
```

#### 9.java.io.IOException: There appears to be a gap in the edit log.  We expected txid 1, but got txid 3.

```
namenode进程中出现如下报错信息:
java.io.IOException: There appears to be a gap in the edit log.  We expected txid 1, but got txid 3

原因：namenode元数据被破坏，需要修复
解决：恢复一下namenode
hadoop namenode -recover
一路选择c，一般就OK了
```
