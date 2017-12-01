---
title: '大数据开发技术知识点'
date: 2017-12-01 16:44:10
tags: bigdata
---


#### CAP理论
```
1) 理论概述
分布式领域中CAP理论：
① C：Consistency，一致性，数据一致更新，所有数据变动都是同步的。
② A：Availability，可用性，系统具有好的响应性能。
③ P：Partition tolerance，分区容错性。以实际效果而言，分区相当于对通信的时限要求。系统如果不能在时限内达成数据一致性，就意味着发生了分区的情况，必须就当前操作在C和A之间做出选择，也就是说无论任何消息丢失，系统都可用。

该理论已被证明：任何分布式系统只可同时满足两点，无法三者兼顾。 因此，将精力浪费在思考如何设计能满足三者的完美系统上是愚钝的，应该根据应用场景进行适当取舍。

2) 一致性分类
一致性是指从系统外部读取系统内部的数据时，在一定约束条件下相同，即数据变动在系统内部各节点应该是同步的。根据一致性的强弱程度不同，可以将一致性级别分为如下几种：
① 强一致性（strong consistency）。任何时刻，任何用户都能读取到最近一次成功更新的数据。
② 单调一致性（monotonic consistency）。任何时刻，任何用户一旦读到某个数据在某次更新后的值，那么就不会再读到比这个值更旧的值。也就是说，可获取的数据顺序必是单调递增的。
③ 会话一致性（session consistency）。任何用户在某次会话中，一旦读到某个数据在某次更新后的值，那么在本次会话中就不会再读到比这个值更旧的值。会话一致性是在单调一致性的基础上进一步放松约束，只保证单个用户单个会话内的单调性，在不同用户或同一用户不同会话间则没有保障。
④ 最终一致性（eventual consistency）。用户只能读到某次更新后的值，但系统保证数据将最终达到完全一致的状态，只是所需时间不能保障。
⑤ 弱一致性（weak consistency）。用户无法在确定时间内读到最新更新的值。
```
<!--more-->
#### MapReduce
```
MapReduce主要包括JobClient、JobTracker、TaskTracker、HDFS四个独立的部分。

1、JobClient
配置参数Configuration，并打包成jar文件存储在HDFS上，将文件路径提交给JobTracker的master服务，然后由master创建每个task将它们分发到各个TaskTracker服务中去执行。

2、JobTracker
这是一个master服务，程序启动后，JobTracker负责资源监控和作业调度。JobTracker监控所有的TaskTracker和job的健康状况，一旦发生失败，即将之转移到其他节点上，同时JobTracker会跟踪任务的执行进度、资源使用量等信息，并将这些信息告诉任务调度器，而调度器会在资源出现空闲时，选择合适的任务使用这些资源。在Hadoop 中，任务调度器是一个可插拔的模块，用户可以根据自己的需要设计相应的调度器。

3、TaskTracker
运行在多个节点上的slaver服务。TaskTracker主动与JobTracker通信接受作业，并负责直接执行每个任务。TaskTracker 会周期性地通过Heartbeat 将本节点上资源的使用情况和任务的运行进度汇报给JobTracker，同时接收JobTracker 发送过来的命令并执行相应的操作（如启动新任务、杀死任务等）。TaskTracker 使用“slot”等量划分本节点上的资源量。“slot”代表计算资源（CPU、内存等）。一个Task 获取到一个slot 后才有机会运行，而Hadoop 调度器的作用就是将各个TaskTracker 上的空闲slot 分配给Task 使用。slot 分为Map slot 和Reduce slot 两种，分别供MapTask 和Reduce Task 使用。TaskTracker 通过slot 数目（可配置参数）限定Task 的并发度。
Task分为Map Task和Reduce Task两种，均由TaskTracker启动。HDFS以block块存储数据，mapreduce处理的最小数据单位为split。
　　
4. HDFS
保存数据和配置信息等。
```

#### ZooKeeper
```
设计目的
1.最终一致性：client不论连接到哪个Server，展示给它都是同一个视图，这是zookeeper最重要的性能。
2 .可靠性：具有简单、健壮、良好的性能，如果消息m被到一台服务器接受，那么它将被所有的服务器接受。
3 .实时性：Zookeeper保证客户端将在一个时间间隔范围内获得服务器的更新信息，或者服务器失效的信息。但由于网络延时等原因，Zookeeper不能保证两个客户端能同时得到刚更新的数据，如果需要最新数据，应该在读数据之前调用sync()接口。
4 .等待无关（wait-free）：慢的或者失效的client不得干预快速的client的请求，使得每个client都能有效的等待。
5.原子性：更新只能成功或者失败，没有中间状态。
6 .顺序性：包括全局有序和偏序两种：全局有序是指如果在一台服务器上消息a在消息b前发布，则在所有Server上消息a都将在消息b前被发布；偏序是指如果一个消息b在消息a后被同一个发送者发布，a必将排在b前面。

工作原理
ZAB: ZooKeeper Atomic Broadcast zk原子广播协议
Zookeeper的核心是原子广播机制，这个机制保证了各个server之间的同步。实现这个机制的协议叫做Zab协议。Zab协议有两种模式，它们分别是恢复模式和广播模式。
1) 恢复模式
当服务启动或者在领导者崩溃后，Zab就进入了恢复模式，当领导者被选举出来，且大多数server完成了和leader的状态同步以后，恢复模式就结束了。状态同步保证了leader和server具有相同的系统状态。
2) 广播模式
一旦Leader已经和多数的Follower进行了状态同步后，他就可以开始广播消息了，即进入广播状态。这时候当一个Server加入ZooKeeper服务中，它会在恢复模式下启动，发现Leader，并和Leader进行状态同步。待到同步结束，它也参与消息广播。ZooKeeper服务一直维持在Broadcast状态，直到Leader崩溃了或者Leader失去了大部分的Followers支持。
```

#### HDFS
```
一个高可靠，高吞吐量的分布式文件系统，通俗的说就是把文件存储到多台服务器上

Namenode全权管理数据块的复制，它周期性地从集群中的每个Datanode接收心跳信号和块状态报告(Blockreport)。接收到心跳信号意味着该Datanode节点工作正常
。块状态报告包含了一个该Datanode上所有数据块的列表。

Namenode是一个中心服务器:单一节点（简化系统的设计和实现），负责管理文件系统的名字空间(namespace)以及客户端对文件的访问。
SecondaryNameNode 用来监控HDFS状态的辅助后台程序，每隔一段时间获取HDFS元数据的快照。
Datanode一个数据块在DataNode以文件存储在磁盘上，包括两个文件，一个是数据本身，一个是元数据包括数据块的长度，块数据的校验和，以及时间戳DataNode启动后向NameNode注册，通过后，周期性（1小时）的向NameNode上报所有的块信息

文件操作:NameNode负责文件元数据的操作，DataNode负责处理文件内容的读写请求，跟文件内容相关的数据流不经过NameNode，只会询问它跟那个DataNode联系，否则NameNode会成为系统的瓶颈。

副本存放位置：DataNode上由NameNode来控制，根据全局情况做出块放置决定，读取文件时NameNode尽量让用户先读取最近的副本，降低带块消耗和读取时延
```

#### spark

- spark数据倾斜时如何优化处理？
```
在shuffle阶段会出现数据倾斜，distinct/groupbyKey/reducebykey/join/repartition时会导致这些情况。
解决方案：
1）预处理数据，将发生数据倾斜提前解决，一般一天处理一次；对于频繁调度spark的场景使用。
2）过滤导致倾斜的key：如果这些key对统计分析结果不重要，那么就过滤掉。
3）阶段聚合：（局部聚合+全局聚合）
4）将reduce join转为map join：适合大表对小表场景，将小表进行广播
5）采样倾斜key并分拆join操作： 对导致倾斜的key加上随机前缀，将rdd打散到多个task中
6）多种方法组合
```

- 日志系统的etl环节具体每步都做了哪些事情？数据质量是如何保证的？
```
数据采集：从不同的数据来源收集日志，如nginx访问日志，应用日志，用户上报日志等
数据清洗：过滤一些无效日志，重复日志等
数据转换：将原始数据转换到适合数据挖掘分析的形式
数据存储：将格式化后的日志存储到文件系统或数据库中，如hdfs、hbase
数据分析：通过大数据技术进行数据挖掘分析，得到某些维度或分类数据结果
数据展示：分析后的数据存储于某数据库，通过报表的形式展现给需求方
```

#### hbase
```
hbase 的特点是什么
(1) Hbase一个分布式的基于列式存储的数据库,基于Hadoop的hdfs存储，zookeeper进行管理。
(2) Hbase适合存储半结构化或非结构化数据，对于数据结构字段不够确定或者杂乱无章很难按一个概念去抽取的数据。
(3) Hbase为null的记录不会被存储.
(4)基于的表包含rowkey，时间戳，和列族。新写入数据时，时间戳更新，同时可以查询到以前的版本.
(5) hbase是主从架构。hmaster作为主节点，hregionserver作为从节点。

Hbase和hive 有什么区别
Hive和Hbase是两种基于Hadoop的不同技术 - -Hive是一种类SQL 的引擎，并且运行MapReduce 任务，Hbase 是一种在Hadoop之上的NoSQL的Key/vale数据库。当然，这两种工具是可以同时使用的。就像用Google 来搜索，用FaceBook 进行社交一样，Hive 可以用来进行统计查询，HBase 可以用来进行实时查询，数据也可以从Hive 写到Hbase，设置再从Hbase 写回Hive。
Hbase非常适合用来进行大数据的实时查询。Facebook用Hbase 进行消息和实时的分析。它也可以用来统计Facebook的连接数。

描述Hbase的rowKey的设计原则.
Rowkey长度原则
Rowkey 是一个二进制码流，Rowkey 的长度被很多开发者建议说设计在10~100 个字节，
不过建议是越短越好，不要超过16 个字节。
Rowkey散列原则
如果Rowkey 是按时间戳的方式递增，不要将时间放在二进制码的前面，建议将Rowkey
的高位作为散列字段，由程序循环生成，低位放时间字段，这样将提高数据均衡分布在每个
Regionserver 实现负载均衡的几率。
Rowkey唯一原则
必须在设计上保证其唯一性。

请描述如何解决Hbase中region太小和region太大带来的冲突.
Region过大会发生多次compaction，将数据读一遍并重写一遍到hdfs 上，占用io，region过小会造成多次split，region 会下线，影响访问服务，调整hbase.hregion.max.filesize 为256m.

简述 HBASE中compact用途是什么，什么时候触发，分为哪两种,有什么区别，有哪些相关配置参数？
在hbase中每当有memstore数据flush到磁盘之后，就形成一个storefile，当storeFile的数量达到一定程度后，就需要将 storefile 文件来进行 compaction 操作。
Compact 的作用：
1>.合并文件
2>.清除过期，多余版本的数据
3>.提高读写数据的效率
HBase 中实现了两种 compaction 的方式：minor and major. 这两种 compaction 方式的区别是：
1、Minor 操作只用来做部分文件的合并操作以及包括 minVersion=0 并且设置 ttl 的过
期版本清理，不做任何删除数据、多版本数据的清理工作。
2、Major 操作是对 Region 下的HStore下的所有StoreFile执行合并操作，最终的结果是整理合并出一个文件。
```


#### flume
```
flume-ng中最重要的核心三大组件是source, channel, sink

source负责从源端收集数据，产出event：
Source是数据源的总称，我们往往设定好源后，数据将源源不断的被抓取或者被推送。
常见的数据源有：ExecSource，KafkaSource，HttpSource，NetcatSource，JmsSource，AvroSource等等。

channel负责暂存event，以备下游取走消费：
Channel用于连接Source和Sink，Source将日志信息发送到Channel，Sink从Channel消费日志信息；Channel是中转日志信息的一个临时存储，保存有Source组件传递过来的日志信息。

sink负责消费通道中的event，写到最终的输出端上：
Sink在设置存储数据时，可以向文件系统中，数据库中，hadoop中储数据，在日志数据较少时，可以将数据存储在文件系中，并且设定一定的时间间隔保存数据。在日志数据较多时，可以将相应的日志数据存储到Hadoop中，便于日后进行相应的数据分析。
```

#### kafka
```
Kafka是最初由Linkedin公司开发，是一个分布式、分区的、多副本的、多订阅者，基于zookeeper协调的分布式日志系统(也可以当做MQ系统)，常见可以用于web/nginx日志、访问日志，消息服务等等，Linkedin于2010年贡献给了Apache基金会并成为顶级开源项目。

Kafka部分名词解释如下：
Broker：消息中间件处理结点，一个Kafka节点就是一个broker，多个broker可以组成一个Kafka集群。
Topic：一类消息，例如page view日志、click日志等都可以以topic的形式存在，Kafka集群能够同时负责多个topic的分发。
Partition：topic物理上的分组，一个topic可以分为多个partition，每个partition是一个有序的队列。
Segment：partition物理上由多个segment组成，下面2.2和2.3有详细说明。
offset：每个partition都由一系列有序的、不可变的消息组成，这些消息被连续的追加到partition中。partition中的每个消息都有一个连续的序列号叫做offset,用于partition唯一标识一条消息.

分析过程分为以下4个步骤：
topic中partition存储分布
partiton中文件存储方式
partiton中segment文件存储结构
在partition中如何通过offset查找message

Kafka高效文件存储设计特点
Kafka把topic中一个parition大文件分成多个小文件段，通过多个小文件段，就容易定期清除或删除已经消费完文件，减少磁盘占用。
通过索引信息可以快速定位message和确定response的最大大小。
通过index元数据全部映射到memory，可以避免segment file的IO磁盘操作。
通过索引文件稀疏存储，可以大幅降低index文件元数据占用空间大小。
```

#### streaming analysis
```
Storm&SparkStreaming
处理模型,延迟
虽然这两个框架都提供可扩展性和容错性,它们根本的区别在于他们的处理模型。而Storm处理的是每次传入的一个事件，而Spark Streaming是处理某个时间段窗口内的事件流。因此,Storm处理一个事件可以达到秒内的延迟，而Spark Streaming则有几秒钟的延迟。

容错、数据保证
在容错数据保证方面的权衡是，Spark Streaming提供了更好的支持容错状态计算。在Storm中,每个单独的记录当它通过系统时必须被跟踪，所以Storm能够至少保证每个记录将被处理一次，但是在从错误中恢复过来时候允许出现重复记录。这意味着可变状态可能不正确地被更新两次。

另一方面，Spark Streaming只需要在批级别进行跟踪处理，因此可以有效地保证每个mini-batch将完全被处理一次，即便一个节点发生故障。(实际上,Storm的 Trident library库也提供了完全一次处理。但是,它依赖于事务更新状态,这比较慢,通常必须由用户实现。)

简而言之,如果你需要秒内的延迟，Storm是一个不错的选择，而且没有数据丢失。如果你需要有状态的计算，而且要完全保证每个事件只被处理一次，Spark Streaming则更好。Spark Streaming编程逻辑也可能更容易，因为它类似于批处理程序(Hadoop)，特别是在你使用批次(尽管是很小的)时。

实现,编程api
Storm初次是由Clojure实现，而 Spark Streaming是使用Scala. 如果你想看看代码还是让自己的定制时需要注意的地方，这样以便发现每个系统是如何工作的。Storm是由BackType和Twitter开发; Spark Streaming是在加州大学伯克利分校开发的。

Storm 有一个Java API, 也支持其他语言，而Spark Streaming是以Scala编程，当然也支持Java

Spark Streaming一个好的特性是其运行在Spark上. 这样你能够你编写批处理的同样代码，这就不需要编写单独的代码来处理实时流数据和历史数据。

产品支持
Storm已经发布几年了，在Twitter从2011年运行至今，同时也有其他公司使用，而Spark Streaming是一个新的项目，它从2013年在Sharethrough有一个项目运行。

Hadoop支持
Storm是一个 Hortonworks Hadoop数据平台上的流解决方案，而Spark Streaming有 MapR的版本还有Cloudera的企业数据平台，Databricks也提供Spark支持。

集群管理集成
尽管两个系统都运行在它们自己的集群上，Storm也能运行在Mesos, 而Spark Streaming能运行在YARN 和 Mesos上。
```

#### ELK
```
https://www.ibm.com/developerworks/cn/opensource/os-cn-elk/

Elasticsearch
Elasticsearch 是一个实时的分布式搜索和分析引擎，它可以用于全文搜索，结构化搜索以及分析。它是一个建立在全文搜索引擎 Apache Lucene 基础上的搜索引擎，使用 Java 语言编写。目前，最新的版本是 2.1.0。
主要特点
实时分析
分布式实时文件存储，并将每一个字段都编入索引
文档导向，所有的对象全部是文档
高可用性，易扩展，支持集群（Cluster）、分片和复制（Shards 和 Replicas）。见图 2 和图 3
接口友好，支持 JSON

Logstash
Logstash 是一个具有实时渠道能力的数据收集引擎。使用 JRuby 语言编写。其作者是世界著名的运维工程师乔丹西塞 (JordanSissel)。
主要特点
几乎可以访问任何数据
可以和多种外部应用结合
支持弹性扩展
它由三个主要部分组成，见图 4：
Shipper－发送日志数据
Broker－收集数据，缺省内置 Redis
Indexer－数据写入

Kibana
Kibana 是一款基于 Apache 开源协议，使用 JavaScript 语言编写，为 Elasticsearch 提供分析和可视化的 Web 平台。它可以在 Elasticsearch 的索引中查找，交互数据，并生成各种维度的表图。
```

#### machine learning
```
http://blog.jobbole.com/108395/
http://blog.jobbole.com/92021/

广义来说，有三种机器学习算法

1、 监督式学习

工作机制：这个算法由一个目标变量或结果变量（或因变量）组成。这些变量由已知的一系列预示变量（自变量）预测而来。利用这一系列变量，我们生成一个将输入值映射到期望输出值的函数。这个训练过程会一直持续，直到模型在训练数据上获得期望的精确度。监督式学习的例子有：回归、决策树、随机森林、K – 近邻算法、逻辑回归等。

2、非监督式学习

工作机制：在这个算法中，没有任何目标变量或结果变量要预测或估计。这个算法用在不同的组内聚类分析。这种分析方式被广泛地用来细分客户，根据干预的方式分为不同的用户组。非监督式学习的例子有：关联算法和 K – 均值算法。

3、强化学习

工作机制：这个算法训练机器进行决策。它是这样工作的：机器被放在一个能让它通过反复试错来训练自己的环境中。机器从过去的经验中进行学习，并且尝试利用了解最透彻的知识作出精确的商业判断。 强化学习的例子有马尔可夫决策过程。

常见机器学习算法名单

这里是一个常用的机器学习算法名单。这些算法几乎可以用在所有的数据问题上：

线性回归
逻辑回归
决策树
SVM
朴素贝叶斯
K最近邻算法
K均值算法
随机森林算法
降维算法
Gradient Boost 和 Adaboost 算法


可以先说说主要的一些概念， 然后挑一个比较熟悉的算法来讲， 包括算法原理、使用场景和缺点等
```


#### 大数据处理相关面试题

- TopK问题
求最大K个数用最小堆，最小K个数用最大堆
- 大数据处理常用排序
快速排序/堆排序/归并排序/桶排序
- 有a、b两个文件，各存放50亿个url，每个url各占64byte，内存限制是4G，如何找出a、b文件共同的url？
- 有一个1G大小的一个文件，里面每一行是一个词，词的大小不超过16字节，内存限制大小是1M，要求返回频数最高的100个词。
- 现有海量日志数据保存在一个超级大的文件中，该文件无法直接读入内存，要求从中提取某天出访问百度次数最多的那个IP。
- 现有海量日志数据保存在一个超级大的文件中，该文件无法直接读入内存，要求从中提取某天出访问百度次数最多的那个IP；假设这个IP总会超过总数的一半。
