---
title: "《离线和实时大数据开发实战》读书笔记"
date: 2019-01-05 08:20:52
excerpt: "《离线和实时大数据开发实战》读书笔记"
tags: read
---

本文是对于《离线和实时大数据开发实战》一书的笔记，在大数据处理这块，我接触更多的是自研或者使用厂内自研的基础工具，虽然思想总是有共通的地方，但是终究缺乏开源实战经验。这是最开始想看这本书的初衷，整个看了一遍下来，工厂环境实战的地方不算多，但是各种简单用例例如 WordCount，非常适合入门。同时，书里系统性的介绍了数据处理的各项开源技术，对之前各种陌生的名词，都能有一个全面的理解。

这篇文章，就是读书过程中的一些摘抄，结合自己一些粗浅的经验，希望对读者有用。

## 1. 数据大图

数据从产生到进入数据平台中被消费和使用，包含四大主要过程：数据产生、数据采集和传输、数据存储和管理以及数据应用，每个过程都需要很多相关数据技术支撑。了解这些关键环节和过程以及支撑它们的关键技术，对一个数据从业者来说，是基本的素养要求。

![数据流程](/assets/images/dataflow/shujuliucheng.jpeg)

随着Google关于分布式计算三篇论文的发表内容主体分别是分布式文件系统Google File System，分布式计算框架MapReduce，分布式数据库Bigtable）和基于Google三篇论文开源实现的Hadoop生态系统（分别对应Google三篇论文——HDFS, MapReduce, HBase）兴起，大数据时代真正到来。

图中流处理部分，目前最为流行和使用广泛的流计算框架是Storm/类Storm的流处理框架和Spark生态的Spark Streaming等。

流程里对应的主要开源技术：

![主要开源技术](/assets/images/dataflow/主要开源技术.jpeg)

当然，实际情况远远不止如此，百家争鸣。

### 1.1. 数据采集和传输

离线批处理目前比较有名和常用的工具是Sqoop，下游的用户主要是离线数据处理平台（如Hive等）。  
实时数据采集和传输最为常用的则是Flume和Kafka，其下游用户一般是实时流处理平台，如Storm、Spark、Flink等。

### 1.2. 数据处理

数据处理是数据开源技术最为百花齐放的领域，离线和准实时的工具主要包括MapReduce、Hive和Spark，流处理的工具主要包含Storm，还有最近较为火爆的Flink、Beam等。

MapReduce将处理大数据的能力赋予了普通开发人员，而Hive进一步将处理和分析大数据的能力赋予了实际的数据使用人员（数据开发工程师、数据分析师、算法工程师和业务分析人员等）。

Spark除了提供了类Hive的SQL接口外，还有用于处理实时数据的流计算框架Spark Streaming，其基本原理是将实时流数据分成小的时间片断（秒或者几百毫秒），以类似Spark离线批处理的方式来处理这小部分数据。

Storm对于实时计算的意义相当于Hadoop对于批处理的意义。Hadoop提供了Map和Reduce原语，使对数据进行批处理变得非常简单和优美。同样，Storm也对数据的实时计算提供了简单的Spout和Bolt原语。Storm集群表面上看和Hadoop集群非常像，但在Hadoop上面运行的是MapReduce的Job，而在Storm上面运行的是Topology（拓扑），Storm拓扑任务和Hadoop MapReduce任务一个非常关键的区别在于：1个MapReduce Job最终会结束，而1个Topology永远运行（除非显式地杀掉它），所以实际上Storm等实时任务的资源使用相比离线MapReduce任务等要大很多，因为离线任务运行完就释放掉所使用的计算、内存等资源，而Storm等实时任务必须一直占用直到被显式地杀掉。

Apache Flink是一个同时面向分布式实时流处理和批量数据处理的开源计算平台，它能够基于同一个Flink运行时（Flink Runtime），提供支持流处理和批处理两种类型应用的功能。

Apache Beam项目重点在于数据处理的编程范式和接口定义，并不涉及具体执行引擎的实现。Apache Beam希望基于Beam开发的数据处理程序可以执行在任意的分布式计算引擎上！G厂出品的项目，总是优秀到总能让人觉得能够一统江湖。

### 1.3. 数据存储

HDFS HBase

### 1.4. 数据应用


## 2. 数据平台大图

### 2.1. 离线数据平台架构

![离线数据平台架构](/assets/images/dataflow/离线数据平台架构.jpeg)

数据仓库技术上，主要有 OLTP 和 OLAP.

OLTP的全称是Online Transaction Processing，顾名思义，OLTP数据库主要用来进行事务处理，比如新增一个订单、修改一个订单、查询一个订单和作废一个订单等。OLTP数据库最核心的需求是单条记录的高效快速处理，索引技术、分库分表等最根本的诉求就是解决此问题。

OLAP数据库本身就能够处理和统计大量的数据，而且不像OLTP数据库需要考虑数据的增删改查和并发锁控制等。OLAP数据一般只需要处理数据查询请求，数据都是批量导入的，因此通过列存储、列压缩和位图索引等技术可以大大加快响应请求的速度。

![OLTP_vs_OLAP](/assets/images/dataflow/OLTP_vs_OLAP.png)

### 2.2. 实时数据平台架构

![实时数据平台架构](/assets/images/dataflow/实时数据平台架构.jpeg)

实时数据采集（如Flume），消息中间件（如Kafka）、流计算框架（如Strom、Spark、Flink和Beam等），以及实时数据存储（如列族存储的HBase）

数据来源通常可以分为两类：数据库和日志文件。对于前者，业界的最佳实践并不是直接访问数据库抽取数据，而是会直接采集数据库变更日志。最近在archsummit大会上听了很多架构分享，都是直接使用binlog。

流计算相对于离线批处理，有几个典型特征；

1. 无边界：数据源源不断  
2. 触发：何时对数据开始计算  
3. 延迟：要求低  
4. 历史数据：Hadoop离线任务如果发现历史某天的数据有问题，通常很容易修复问题而且重运行任务，但是对于流计算任务来说基本不可能或者代价非常大，因为首先实时流消息通常不会保存很久（一般几天），而且保存历史的完全现场基本不可能。  

流计算框架现在最为百花齐放：Storm是最早的流计算技术和框架，也是目前最广为所知的实时数据处理技术，但是实际上还有其他的开源流计算技术，如Storm Trident、Spark Streaming、Samza、Flink、Beam等，商业性的技术还有Google MillWheel和亚马逊的Kinesis等。

![流计算框架对比](/assets/images/dataflow/流计算技术.jpeg)

### 2.3. 数据管理

数据仓库的数据集成也叫ETL（抽取：extract、转换：transform、加载：load），是数据平台构建的核心，也是数据平台构建过程中花费时间最多、花费精力最多的阶段。ETL泛指将数据从数据源头抽取、经过清洗、转换、关联等转换，并最终按照预先设计的数据模型将数据加载到数据仓库的过程。

## 3. 离线数据开发：大数据开发的主战场

### 3.1. Hadoop

Hadoop是一个蓬勃发展的生态，从底层调度和资源管理的YARN/ZooKeeper到SQL on Hadoop的Hive，从分布式的NoSQL数据库HBase到流计算Storm框架，从海量日志采集处理框架Flume到海量消息分布式订阅-消费系统Kafka，所有这些技术共同组成了一个完善的、彼此良性互动和补充的Hadoop大数据生态系统。

#### 3.1.1 HDFS

优势：

+ 处理超大文件  
+ 运行于廉价的商用机器集群上  
+ 高容错性和高可靠性  
+ 流式的访问数据（一次写入、多次读取）  

劣势：

+ 不适合低延迟数据访问：HDFS是为了处理大型数据集而设计的，主要是为达到高的数据吞吐量而设计的，延迟时间通常实在分钟乃至小时级别  
+ 无法高效存储大量小文件  
+ 不支持多用户写入和随机文件修改  

![hdfs_arch](/assets/images/dataflow/hdfs_arch.png)

#### 3.1.2 MapReduce

优势：
+ 易于编程  
+ 良好的扩展性  
+ 高容错性  

劣势：
+ 实时计算  
+ 流计算  
+ DAG（有向图）计算：多个应用程序存在依赖关系，后一个应用程序的输入为前一个的输出。在这种情况下，MapReduce并不是不能做，而是使用后，每个MapReduce作业的输出结果都会写入磁盘，会造成大量的磁盘IO，导致性能非常低下，此时可以考虑用Spark等迭代计算框架。

MapReduce架构如图：

![mr_arch](/assets/images/dataflow/mr_arch.png)

MapReduce 运行 WordCount 的例子：

![mr_word_count](/assets/images/dataflow/mr_word_count.png)

总体来说，Shuffle阶段包含在Map和Reduce两个阶段中，在Map阶段的Shuffle阶段是对Map的结果进行分区（partition）、排序（sort）和分隔（spilt），然后将属于同一个分区的输出合并在一起（merge）并写在磁盘上，同时按照不同的分区划分发送给对应的Reduce（Map输出的划分和Reduce任务的对应关系由JobTracker确定）的整个过程；Reduce阶段的Shuffle又会将各个Map输出的同一个分区划分的输出进行合并，然后对合并的结果进行排序，最后交给Reduce处理的整个过程。

### 3.2. Hive

Hive是建立在Hadoop体系架构上的一层SQL抽象，使得数据相关人员使用他们最为熟悉的SQL语言就可以进行海量数据的处理、分析和统计工作，而不是必须掌握Java等编程语言和具备开发MapReduce程序的能力。

![hive](/assets/images/dataflow/hive.png)

Hive join表的例子：

![hive_join](/assets/images/dataflow/hive_join.png)

此外还有Impala、Drill、HAWQ、Presto、Dremel等SQL on Hadoop的技术。

值得一提的是：Apache Drill是Dremel的开源实现。
Google于2010年在《Dremel: Interactive Analysis of WebScaleDatasets》一文中公开了Dremel的设计原理

![sql_on_hadoop](/assets/images/dataflow/sql_on_hadoop.png)

数据处理不免遇到数据倾斜，即热点问题：

**“数据量大”从来都不是问题，因为理论上来说，都可以通过增加并发的节点数来解决。
但是如果数据倾斜或者分布不均了，那么就会是问题。此时不能简单地通过增加并发节点数来解决问题，而必须采用针对性的措施和优化方案来解决。**

![hot_spot](/assets/images/dataflow/hot_spot.jpeg)

在数据处理时，这类问题真正难解，我在实际项目里负责过多个“单点”模块，虽然极力把实例做成了无状态的，但是跟数据倾斜的问题一直在斗争着。

### 3.3. 维度建模

这块主要是先深入理解业务，然后根据各种方法论建模。

“数据湖”（data lake）的概念最早是在2011年福布斯的一篇文章《Big Data Requires a big new Architecture》中提出的。该文章认为，在大数据时代，数据量的庞大、数据来源和类型的多元化、数据价值密度低、数据增长快速等特性使得传统的数据仓库无法承载，因此需要一个新的架构作为大数据的支撑，而这种架构即是数据湖。

![data_lake](/assets/images/dataflow/data_lake.png)

跟传统的数据仓库的核心区别：

1. 数据湖存放所有数据  
2. schema-on-read  
3. 更容易适应变化  
4. 更快的洞悉能力  

总的来说，数据湖collect everything，然后鼓励用户自助、自由分析数据。

## 4. 实时数据开发：大数据开发的未来

离线处理的模式可以满足“看”数据的需求，但是在大数据时代，数据已经不仅局限在“看”。在很多场景下，需要在数据产生的那个时刻立刻捕获数据，并进行针对性的业务动作（如广告推送、实时商品推荐、实时新闻推荐等）。

[阿里巴巴中台战略思想与架构实践笔记](https://izualzhy.cn/alibaba-reading#6-%E6%89%93%E9%80%A0%E6%95%B0%E5%AD%97%E5%8C%96%E8%BF%90%E8%90%A5%E8%83%BD%E5%8A%9B)里也提到了，对于时效性更敏感的业务需求，我们需要有实时统计计算的能力。

### 4.1. Storm

目前有很多专业的流计算处理工具和框架，较为知名的包括Apache Storm、Spark Streaming、LinkIn Samza、Apache Flink和Google MillWheel等，但是其中最广为人所知的无疑是Storm，这既跟Storm是最早流行的一个流计算技术有关，但是也和Storm本身的诸多优点有关。

尽管Storm的开源者Twitter已经放弃了Storm，转向其下一代流计算引擎Heron，但是作为第一代流计算引擎，Storm其提出的流计算的概念、架构、并发配置以及核心API等仍然被后续的流计算技术广为采纳。同时对于极低延迟要求的某些场景，Storm仍然是很好的解决方案。

因此，这一节介绍的概念可能有点零碎，但是在接下来的其他开源技术对比时，就会比较好的串联起来。

Hadoop提供了Map和Reduce原语，使得对数据进行批处理变得非常简单和优美。同样，Storm也对数据的实时计算提供了简单的spout和bolt原语。Storm集群表面上看和Hadoop集群非常像，但Hadoop上面运行的是MapReduce的Job，而Storm上面运行的是topology（拓扑），它们非常不一样，比如一个MapReduce Job最终会结束，而一个Storm topology永远运行（除非显式地杀掉它）。

Storm集群架构：

![storm_arch](/assets/images/dataflow/storm_arch.png)

Storm关键概念：

1. topology:一个实时应用程序  
2. tuple:处理的基本消息单元  
3. 流：一个流由无数个元组序列构成，这些元组并行、分布式的被创建和执行  
4. spout: topology的流的来源，是一个topology中产生源数据流的组件  
5. blot：拓扑中的所有处理逻辑都在bolt中完成  
6. 流分组：(stream grouping)用来定义一个Stream应该如何分配数据给bolts上的多个任务  

关于流分组，Storm定义的8种内置数据流分组方式如下。
❏ shuffle grouping（随机分组）：这种方式会随机分发tuple给bolt的各个任务，每个bolt实例接收到相同数量的tuple。  
❏ fields grouping（按字段分组）：根据指定字段的值进行分组，例如，一个数据流根据“word”字段进行分组，所有具有相同“word”字段值的（tuple）会路由到同一个（bolt）的task中。  
❏ all grouping（全复制分组）：将所有的tuple复制后分发给所有bolt task，每个订阅数据流的task都会接收到所有tuple的一份备份。  
❏ globle grouping（全局分组）：这种分组方式将所有的tuples路由到唯一的任务上，Storm按照最小的task ID来选取接收数据的task。注意，当使用全局分组方式时，设置bolt的task并发度是没有意义的（spout并发有意义），因为所有tuple都转发到同一个task上了，此外因为所有的tuple都转发到一个JVM实例上，可能会引起Storm集群中某个JVM或者服务器出现性能瓶颈或崩溃。  
❏ none grouping（不分组）：在功能上和随机分组相同，是为将来预留的。  
❏ direct grouping（指向型分组）：数据源会调用emitDirect（）方法来判断一个tuple应该由哪个Storm组件来接收。  
❏ local or shuffle grouping（本地或随机分组）：和随机分组类似，但是会将tuple分发给同一个worker内的bolt task（如果worker内有接收数据的bolt task），其他情况下则采用随机分组的方式。本地或随机分组取决于topology的并发度，可以减少网络传输，从而提高topology性能。  
❏ partial key grouping：与按字段分组类似，根据指定字段的一部分进行分组分发，能够很好地实现负载均衡，将元组发送给下游的bolt对应的任务，特别是在存在数据倾斜的场景下，使用partial key grouping能够更好地提高资源利用率。  

Storm计算WordCount的例子：

![storm_word_count_instance](/assets/images/dataflow/storm_word_count_instance.png)

介绍Trident的引入之前，首先介绍流计算的三种语义：at most once（至多一次）、at least once（至少一次）以及exactly once（恰好一次）。  
❏ at most once：保证每个消息会被投递0次或者1次，在这种机制下，消息很有可能会丢失。  
❏ at least once：保证了每个消息会被默认投递多次，至少保证有一次被成功接收，信息可能有重复，但是不会丢失。  
❏ exactly once：意味着每个消息对于接收者而言正好被接收一次，保证即不会丢失也不会重复。  

早期的Storm无法提供exactly once的语义支持，后期Storm引入了Trident高级原语，提供了exactly once的语义支持。

首先介绍流计算中反压（back pressure）的概念。所谓流计算中的反压，指的是流计算job（比如Storm中的一个拓扑）处理数据的速度小于数据流入的速度时的处理机制，通常来说，反压出现的时候，数据会迅速累积，如果处理不当，会导致资源耗尽甚至任务崩溃。

这在流处理过程中非常常见，通常是由于源头数据量突然急剧增加所导致的，比如电商的大促、节日活动等。

新的Storm自动反压机制（Automatic Back Pressure）通过监控bolt中的接收队列的情况来实现，当超过高水位值时，专门的线程会将反压信息写到ZooKeeper, ZooKeeper上的watch会通知该拓扑的所有worker都进入反压状态，最后spout降低tuple发送的速度。

### 4.2. Spark Streaming

Spark最初属于伯克利大学的研究性项目，于2010年正式开源，于2013年成为Apache基金项目，并于2014年成为Apache基金的顶级项目。Spark用了不到5年的时间就成了Apache的顶级项目，目前已经被国内外的众多互联网公司使用，包括Amazon、Ebay、淘宝、腾讯等。

相比于Storm原生的实时处理框架，Spark Streaming基于微批的方案带来了吞吐量的提升，但是也导致了数据处理延迟的增加——基于Spark Streaming的实时数据处理方案的数据延迟通常在秒级甚至分钟级。

传统Hadoop基于MapReduce的方案适用于大多数的离线批处理场景，但是对于实时查询、迭代计算等场景非常不合适，这是由其内在局限决定的。

❏ MapReduce只提供Map和Reduce两个操作，抽象程度低，但是复杂的计算通常需要很多操作，而且操作之间有复杂的依赖关系。  
❏ MapReduce的中间处理结果是放在HDFS文件系统中的，每次的落地和读取都消耗大量的时间和资源。  
❏ 当然，MapReduce也不支持高级数据处理API、DAG（有向无环图）计算、迭代计算等。  

Spark则较好地解决了上述这些问题：
❏ Spark通过引入弹性分布式数据集（RDD）以及RDD丰富的动作操作API，非常好地支持了DAG计算和迭代计算。  
❏ Spark通过内存计算和缓存数据非常好地支持了迭代计算和DAG计算的数据共享，减少了数据读取的IO开销，大大提高了数据处理速度。  
❏ Spark为批处理（Spark Core）、流式处理（Spark Streaming）、交互分析（Spark SQL）、机器学习（MLLib）和图计算（GraphX）提供了一个统一的平台和API，非常便于使用。  

RDD是Spark中最为核心和重要的概念。RDD，全称为Resilient Distributed Dataset，在Spark官方文档中被称为“一个可并行操作的有容错机制的数据集合”，这个听起来有点抽象。实际上，RDD就是一个数据集，而且是分布式的，也就是可分布在不同的机器上，同时Spark还对这个分布式数据集提供了丰富的数据操作以及容错性等。

![spark_circle](/assets/images/dataflow/spark_circle.jpeg)

Spark Streaming中数据处理的单位是一批而不是一条，Spark会等采集的源头数据累积到设置的间隔条件后，对数据进行统一的微批处理。这个间隔是Spark Streaming中的核心概念和关键参数，直接决定了Spark Streaming作业的数据处理延迟，当然也决定着数据处理的吞吐量和性能。

Spark Streaming中基本的抽象是离散流（即DStream）。DStream代表一个连续的数据流。在Spark Streaming内部中，DStream实际上是由一系列连续的RDD组成的。每个RDD包含确定时间间隔内的数据，这些离散的RDD连在一起，共同组成了对应的DStream。

![spark_streaming](/assets/images/dataflow/spark_streaming.png)

### 4.3. Flink

Storm延迟低但是吞吐量小，Spark Streaming吞吐量大但是延迟高，那么是否有一种兼具低延迟和高吞吐量特点的流计算技术呢？答案是有的，就是Flink。

实际上，Flink于2008年作为柏林理工大学的一个研究性项目诞生，但是直到2015年以后才开始逐步得到认可和接受，这和其自身的技术特点契合了大数据对低实时延迟、高吞吐、容错、可靠性、灵活的窗口操作以及状态管理等显著特性分不开，当然也和实时数据越来越得到重视分不开。

阿里巴巴启动了Blink项目，目标是扩展、优化、完善Flink，使其能够应用在阿里巴巴大规模实时计算场景。

Flink几乎具备了流计算所要求的所有特点。  
❏ 高吞吐、低延迟、高性能的流处理。  
❏ 支持带有事件时间的窗口（window）操作。  
❏ 支持有状态计算的exactly once语义。  
❏ 支持高度灵活的窗口操作，支持基于time、count、session以及data-driven的窗口操作。  
❏ 支持具有反压功能的持续流模型。  
❏ 支持基于轻量级分布式快照（snapshot）实现的容错。  
❏ 一个运行时同时支持batch on Streaming处理和Streaming处理。  
❏ Flink在JVM内部实现了自己的内存管理。  
❏ 支持迭代计算。  
❏ 支持程序自动优化：避免特定情况下shuffle、排序等昂贵操作，中间结果有必要时会进行缓存。  

Flink技术栈:

![flink](/assets/images/dataflow/flink.jpeg)

Flink底层用流处理模型来同时处理上述两种数据。在Flink看来，有界数据集不过是无界数据集的一种特例；而Spark Streaming走了完全相反的技术路线，即它把无界数据集分割成了有界的数据集而通过微批的方式来对待流计算。

同Spark Streaming、Storm等流计算引擎一样，Flink的数据处理组件也被分为三类：数据输入（source）、数据处理（transformation）和数据输出（sink）。此外，Flink对数据流的抽象称为Stream（Spark中称为DStream, Storm中也称为Stream）。

Flink支持对各种窗口进行统计，具体如下。  
❏ 时间窗口（time window）：包括翻滚窗口（即不重叠的窗口，比如12点1分到5分是一个窗口，12点6分到10分是一个窗口）、滑动窗口（有重叠的窗口，比如每一分钟统计一下过去5分钟内的指标）。  
❏ 事件窗口（count window）：比如每100个源头数据。  
❏ 会话窗口（session window）：通过不活动的间隙来划分。  

使用Flink计算WordCount的逻辑及物理流程：

![flink_word_count_logical](/assets/images/dataflow/flink_word_count_logical.png)
![flink_word_count_physical](/assets/images/dataflow/flink_word_count_physical.png)

同其他流计算框架一样，Flink也有数据输入、数据处理和数据输出组件，只不过在Flink中，它们分别叫作source组件、transformation组件和sink组件，同时对应于Storm中的拓扑（topology），Flink中称之为Flink Stream dataflow。

#### 4.3.1. Flink关键技术-容错

Flink容错机制的核心是分布式数据流和状态的快照，为了保证失败时从错误中恢复，因此需要对数据对齐。

![flink_barrier](/assets/images/dataflow/flink_barrier.png)

#### 4.3.2. Flink关键技术-存储

Flink采用了单机性能十分优异的RocksDB作为状态的后端存储，但单机是不可靠的，所以Flink还对将单机的状态同步到HDFS上以保证状态的可靠性。另外，对于从RocksDB到HDFS上checkpoint的同步，Flink也支持增量的方式，能够非常好地提高checkpoint的效率，这里不做过多展开。

#### 4.3.3. Flink关键技术-水位线

Flink相比其他流计算技术的一个重要特性是支持基于事件时间（event time）的窗口操作。但是事件时间来自于源头系统，网络延迟、分布式处理以及源头系统等各种原因导致源头数据的事件时间可能是乱序的，即发生晚的事件反而比发生早的事件来得早，或者说某些事件会迟到。Flink参考Google的Cloud Dataflow，引入水印的概念来解决和衡量这种乱序的问题。

其实概念很简单，就是一个时间戳而已，理想情况下处理时间跟事件时间相等最完美，但是实际处理时间一定小于事件时间，

水位线生成最常用的办法是with periodic watermark，其含义是定义一个最大允许乱序的时间，比如某条日志时间为2017-01-01 08:00:10，如果定义最大乱序时间为10s，那么其水位线时间戳就是2017-01-01 08:00:00，其含义就是说8点之前的所有数据都已经到达，那么某个小时窗口此时就可以被触发并计算该小时内的业务指标。

![flink_watermark](/assets/images/dataflow/flink_watermark.png)

解释下窗口机制：例如，某job使用基于事件时间的窗口操作，假定使用5min的翻滚窗口，并且允许延迟1min延迟，那么Flink将在12:00和12:05之间并且当落入此间隔时间戳的第一个元素到达时创建此窗口，并将在watermark超过12:06时将其删除。

#### 4.3.4. Flink关键技术-撤回

在流计算的某些场景下，需要撤回（retract）之前的计算结果进行，Flink提供了撤回机制

#### 4.3.4. Flink关键技术-反压机制

Storm是通过监控process bolt中的接收队列负载情况来处理反压，如果超过高水位值，就将反压信息写到ZooKeeper，由ZooKeeper上的watch通知该拓扑的所有worker都进入反压状态，最后spout停止发送tuple来处理的。而Spark Streaming通过设置属性“spark.streaming.backpressure.enabled”可以自动进行反压处理，它会动态控制数据接收速率来适配集群数据处理能力。对于Flink来说，不需要进行任何的特殊设置，其本身的纯数据流引擎可以非常优雅地处理反压问题。


### 4.4. Beam技术

离线数据处理基本上都基于Hadoop和Hive，那么实时流计算技术能否像离线数据处理一样出现Hadoop和Hive这种事实上的技术标准呢？Google的答案是：可以，这种技术就是Beam。
Apache Beam被认为是继MapReduce、GFS、Bigtable等之后，Google在大数据处理领域对开源社区的又一大贡献。当然，Beam也代表了Google对数据处理领域一统江湖的雄心。

![beam](/assets/images/dataflow/beam.png)

此刻不得不说出那句古话：**天下大势分久必合合久必分。**

Apache Beam本身不是一个流处理平台，而是一个统一的编程框架，它提供了开源的、统一的编程模型，帮助用户创建自己的数据处理流水线，从而可以在任意执行引擎之上运行批处理和流处理任务。

![google_cloud_dataflow](/assets/images/dataflow/google_cloud_dataflow.png)

Beam的特点：

+ 统一  
+ 可移植  
+ 可扩展  

Beam SDK的基本概念也是类似的，也包含了如下基本要素：任务（Beam叫Pipeline、Storm叫拓扑、Flink叫Stream dataflow, Spark没有将其明确抽象出来）、分布式数据集（Beam叫PCollection, Storm叫Stream、Spark叫RDD或DStream、Flink叫DataStream、）输入、转换和输出等。

Beam支持如下完整窗口类型。  
❏ 翻滚窗口。  
❏ 滑动窗口。  
❏ session窗口。  
❏ 全局窗口。  
❏ 基于日历的窗户（暂不支持Python）。  

如同Flink一样，Beam支持通过水印水位线（watermark）来处理迟到的数据。实际上，Beam和Flink都是参考了Google的MillWheel流计算引擎，所以其处理迟到数据的机制非常类似。

![beam_watermark](/assets/images/dataflow/beam_watermark.png)

Beam会结合watermark和触发器来决定何时将计算结果输出，实际中的触发机制需要详细考虑，因为触发太早会丢失一部分数据，丧失精确性，而触发太晚又会导致延迟变长，而且会囤积大量数据。

数据触发上，事件时间触发器基于水位线来触发。当然，Beam也可以基于处理时间触发，例如`AfterProcessingTime.pastFirstElement-InPane()`在接收到数据一定时间后会输出结果。
Beam也支持数据驱动触发器，例如`AfterPane.elementCountAtLeast()`触发器就基于数据计数来触发，比如收集到50个就触发，但是实际中有可能50个也许永远到不了，此时可以综合基于时间的触发和基于计数的触发来解决这个问题。

Beam抽象出了数据处理（包含批处理和实时处理）的通用处理范式——Beam Model，即What（要对数据进行何种操作）、Where（在什么范围内计算）、When（何时输出计算结果）、How（怎么修正迟到数据）。

### 4.5. Stream SQL

![stream_sql](/assets/images/dataflow/stream_sql_arch.png)

阿里云Stream SQL的底层就是Flink引擎（实际是Blink，也就是Alibaba Flink，可以认为Blink是Flink的企业版本，它提供了众多企业级特性，同时也在不停回馈Flink社区）。阿里云Stream SQL的语法基本和Flink SQL一致，而且其提供了流计算SQL完善的开发环境支持（IDE环境）。
