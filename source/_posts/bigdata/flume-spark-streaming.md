---
title: 'Spark Streaming从Flume读取数据流(pull模式)'
date: 2017-10-19 13:51:01
tags: flume spark_streaming
---


#### jar包准备

参考官方文档： http://spark.apache.org/docs/latest/streaming-flume-integration.html

当前测试flume使用到的jar包版本如下:
```
spark-streaming-flume-sink_2.11-2.2.0.jar
scala-library-2.11.8.jar
commons-lang3-3.5.jar
```
这几个jar包下载后放到flume安装目录 `./flume/lib/` 中。

spark streaming用到的jar版本如下:
```
spark-streaming-flume-assembly_2.11-2.2.0.jar
```
在 http://search.maven.org 下载后放到spark jar依赖目录。
<!--more-->

#### flume 配置启动

假设flume数据源为本地日志文件: /tmp/log_source/src.log
新增config文件, 如flume-spark.conf:
```bash
a1.channels = c1
a1.sinks = spark
a1.sources = r1

a1.sinks.spark.type = org.apache.spark.streaming.flume.sink.SparkSink
a1.sinks.spark.hostname = your_hostname
a1.sinks.spark.port = 9999
a1.sinks.spark.channel = c1

a1.sources.r1.type=exec
a1.sources.r1.channels=c1
a1.sources.r1.command=tail -F /tmp/log_source/src.log

a1.channels.c1.type = file
a1.channels.c1.checkpointDir=/tmp/flume-spark/tmp/checkpoint
a1.channels.c1.dataDirs=/tmp/flume-spark/tmp/data
```

启动测试:
```bash
hadoop@1:/usr/local/flume$ bin/flume-ng agent -c conf -f conf/flume-spark.conf -n a1 -Dflume.root.logger=DEBUG,console
```

#### spark streaming 接收数据(python)

词频统计逻辑实现test_streaming.py:
```python
from __future__ import print_function
import sys
import logging

from pyspark import SparkContext
from pyspark.streaming import StreamingContext
from pyspark.streaming.flume import FlumeUtils


def flume_main():
    sc = SparkContext(appName="streaming_analysis_wordcount")
    sc.setLogLevel("WARN")
    ssc = StreamingContext(sc, 1)

    addrs = [("your_hostname",  9999), ]
    fps = FlumeUtils.createPollingStream(ssc, addrs)
    lines = fps.map(lambda x: x[1])
    counts = lines.flatMap(lambda line: line.split(" ")) \
        .map(lambda word: (word, 1)) \
        .reduceByKey(lambda a, b: a+b)
    counts.pprint()

    ssc.start()
    ssc.awaitTermination()    


if __name__ == "__main__":
    flume_main()
```
提交spark任务:
```bash
hadoop@1:/data/test$ /usr/local/spark/bin/spark-submit --master yarn --deploy-mode client test_streaming.py
```
运行后可看到统计输出如:
```bash
-------------------------------------------
Time: 2017-10-19 11:49:00
-------------------------------------------


-------------------------------------------
Time: 2017-10-19 11:49:01
-------------------------------------------
('current', 20)
('20171019', 20)
('11:48:58', 1)
('11:48:59', 10)
('5435854358', 1)
('5435954359', 1)
('5436354363', 1)
('5436454364', 1)
('5436954369', 1)
('5435454354', 1)
...
```

