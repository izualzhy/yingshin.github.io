---
title: "大数据和后端服务"
date: 2024-02-25 06:27:45 
tags: Pensieve
---

大数据和后端服务之间的差别，远比后端服务之间，比如推荐架构、搜索架构、直播架构等的差别要大。两者比较的文章似乎很少见。

但实际上，我在做后端服务的时候，也曾调研过能否使用大数据的组件，比如 Flink、Kafka。很多后端服务也会用到大数据的存储，比如 Hbase 来存储数据。

同时，我刚从后端转到大数据开发时，对各种差别感到疑惑。如今做了几年大数据，有的疑惑逐渐解开，有的疑惑依旧看不清，有必要阶段性的总结一下。当然，工程师不应该限制自己是大数据、前端还是后端还是算法，但是试图理清区别和联系，能够让我们的视野看的更高。 

## 1. 大数据的技术本质还是后端服务

以在大数据离线任务开发中，常见的 Apache DolphinScheduler 工作流调度系统为例。DolphinScheduler 架构是一套典型的去中心化的 Master-Worker 架构。相同的架构，我之前做后端服务时也实现过一版，用来更新搜索词典、数据删除、Key白名单等操作，区别仅仅是总控模块不叫 master 而是 center.

系统的关注点是类似的，比如 master/center 依赖 zk 实现高可用；worker 处理能力线性扩展；RPC的选型、request/response 的数据结构；数据库读写控制在总控模块；数据库的优化等等。

Flink 里的 JobManager、TaskManager 也是如此：JobManager 负责资源管理、Checkpoint、RestEndpoint这些服务，TaskManager 则负责具体执行用户代码。两者之间通过 akka RPC 通信，采用内存队列缓存 RPC 消息。同时心跳机制、HA的实现、服务发现等等，也都是后端服务实现时需要重点考虑的。

```
                      ┌───────────────────────────────┐
                      │                               │
                      │   master/center/jobmanager    │ HA/Service Discovery/Storage/Coordinate/...
                      │                               │
                      └─────┬────────────────────┬────┘
                            │                    │
                            │                    │
     RPC/pool/queue/ACK/... │                    │ RPC/pool/queue/ACK/...
                            │                    │
                            │                    │
┌───────────────────────────┴───┐              ┌─┴─────────────────────────────┐
│                               │              │                               │
│       worker/taskmanager      │              │       worker/taskmanager      │
│                               │              │                               │
└───────────────────────────────┘              └───────────────────────────────┘
```

部署上，大数据的组件 Flink Spark MapReduce 都通过 YARN 部署，后端服务往往通过 K8S 部署。而这几年的发展趋势，大数据这些组件，也支持部署到 K8S 了。   
至于分组隔离、存算分离、冷热存储等，也都屡见不鲜。

**因此，大数据组件的技术，本质上还是后端服务**。

## 2. 大数据的技术使用上更加简单
由于面向不同的用户，大数据的组件往往追求简单的使用方式。

还是以第一节的例子来说。

我在实现 center-worker 这套系统时，重点考虑 center. 至于 worker，则是开放了 RPC 协议，按需实现不同的功能。center 按照业务逻辑顺序，调用不同的 worker。系统面向的用户是 RD，这样的好处，是不同小组的 RD 可以开发不同的 worker，代码、语言甚至都可以不同。worker 的代码变化频繁，每组 worker 单独评估性能、稳定性以及上线。

而在 dolphinscheduler，worker 的能力是完全相同的，不同的需求通过内置的 plugin 实现，例如执行 SparkSQL、DorisSQL、FlinkSQL 等等。worker 的能力也就比较固定，一旦成型代码变化不大。系统面向的用户大部分是数仓，用户关注的是 SQL 执行的结果，至于上一节提到的架构、RPC的协议这些，都不是用户关心的。

**因此，大数据的技术，使用上需要尽量简单**。

这种易用性的追求，也会导致一些常见的错误。以 Flink 的 DataStream API 为例，这段 scala 代码目的是过滤 >threshold 的数据 ：


```scala
object ATestStream extends App {
  val env = StreamExecutionEnvironment.getExecutionEnvironment

  private val threshold = 5

  env.fromSequence(1, 10)
    .filter(_ > threshold)
    .print("after filter:")

  env.execute()
}
```

程序输出的内容，不熟悉的 RD 大概率会认为是`after filter: 6, after filter: 7, ...`.  

但实际上是：
```
after filter:> 1
after filter:> 2
after filter:> 3
after filter:> 4
after filter:> 5
after filter:> 6
after filter:> 7
after filter:> 8
after filter:> 9
after filter:> 10
```

究其原因，是因为 DataStream API 和 MapReduce 一样，都是声明式的：
1. 用户的 main 方法，不是程序的入口，只是 TaskManager 很小的一部分。
2. 用户的 filter/print/... 方法，嵌入在真正的物理执行算子里，用来处理数据。

`threshold` 这个变量，是 main 方法的局部变量，而 filter/print/... 方法，是运行在 TaskManager 上的。进一步的，`main`方法的执行，可能是在 client 端，也可能是在 JobManager.因此在 TaskManager，`threshold`实际是个未初始化的变量。当然，这里也跟是否是 Standalone 运行环境有关，就不在技术层面展开了。 

类似的，在平时为实时计算平台的用户答疑时，也会看到这样出问题的 case:

```scala
   // kafka
    val prop = new Properties()
    prop.setProperty("bootstrap.servers", "...")
    // sasl_ssl 参数
    prop.setProperty("ssl.truststore.location", localFile.getPath)
    // 初始化kafka
    val kafkaConsumer = new FlinkKafkaConsumer[String](topics, new SimpleStringSchema(), prop)
```

但是毫无疑问，这种声明式的使用方式，易用性是非常高的。我看到很多 qps 达到百万的实时任务，用户只需要实现 map/filter/sink 等方法，就可以轻松实现。

这种只需要关注输入输出，不用考虑多线程的竞争关系，也不用考虑词典(大数据称为维表 Temporal Join)是如何加载和查询的，用户只需要在单线程里根据输入实现输出逻辑。我们在封装后端模块时，理想的状态就是这样的。

因此从这点上，大数据的技术是靠前的。基于 K8S 的函数计算，出现的晚，但是使用上，还是这些声明式的 API 更加简单。

对易用性的追求，也是大数据离线、实时计算都支持 SQL 的一个原因，目的还是要简化开发。

## 3. 大数据的技术运维上更加复杂

系统整体的复杂度是守恒的，用户看到的简单，底层的复杂度就会变高。就跟冰山一样，露在海面上的，只是极小的一部分。

比如大数据实时任务的开发者，很多对自己任务的单并发性能并不了解，更别说数据倾斜的影响了。  

这背后也有原因：  
1. 如果我只花了两天时间开发，愿意再去花相同的时间压测么？大部分公司应该都是相同的答案，即使Yes，我面对了几百个任务，要逐个压测？     
2. 如果我压测性能有问题，怎么解决呢？本来我只需要看看 Flink 的 API，现在你要我去了解各类性能指标？分析 Source/Sink/Rebalance？  

因此，**默认值**在大数据里是个非常常见的东西。CPU个数、内存大小都有默认值，用户只写 SQL/Stream API，运行起来有问题，往往需要技术在底层优化，或者给出调整的参数。任务的并发数也是如此，所以在 Spark/Flink 里都会看到资源自动调优 这种概念。类似的事情，在后端开发是绝对禁止的。**默认值**这个东西，使用者感觉不到。但是一旦多了，修改的成本成倍增加，牵一发而动全身，一旦修改，影响全部任务。

再看个 traceid 的例子，假定调用流程为：

```
后端服务：后端模块A  ->  后端模块B  ->  后端模块C  ->  后端模块D  ->  ...

大数据  ：实时任务A  ->  实时任务B  ->  实时任务C  ->  实时任务D  ->  ...
```

两者相似的点都是数据的流转过程，我们想要通过 trace 了解各个模块/任务处理的时间，某个模块/任务是否丟数等。

对于后端模块，可以方便的加入 traceid：

```cpp
// 收到 RPC 数据 module_input
module_input = (meta_data, input_data)

// 处理数据逻辑，只关注 input_data
output_data = f(input_data)

// 补充 medata_data，RPC 发送到下一个模块
module_output = (meta_data, output_data)
```

这样的好处：
1. 业务方只需要实现`f(...)`方法，单线程处理输入输出  
2. traceid 存储到 metadata，对业务不可见，不会困扰也不会误修改  

即使有数据扩散或者主动丢弃，业务方也完全意识不到 meta_data 的存在，专心实现 f 方法处理数据就行，除非也有 trace 日志的需要。

换到 flink 的 DataStream API，假定业务方实现了如下代码：

```scala
sourceStream.map(input_data -> middle_data)
            .keyBy(middle_data_1.key)
            .map(middle_data -> output_data)
            .addSink(sink)
```

无论是修改转换 transformation 的过程，还是添加类似 watermark、latencymark 的标记，都非常复杂。主要是本身已经是声明式的 API，不是后端开发习惯的线程模型。  

当然，对大数据处理过程增加 trace，本身数据量、计算量也都是个挑战。

我觉得大数据的开发非常简单，而对于异常处理、脏数据的处理、case的追查这类运行中的问题，要比后端服务差一些。但是同时，实时任务、高优看板数据的稳定性，是非常高的。很多边界条件的处理，以配置项/DDL properties/声明式 API 的方式暴露给了用户，复杂度留到了 runtime 阶段。因此，**大数据的技术运维上更加复杂**。


## 4. 总结：解决问题

技术的本质是为了解决问题，技术演变的过程也是问题逐步解决的过程。由于高度的封装带来易用性的提高，大数据可以轻松写出百万级别qps、秒级延迟的任务，而之后可能出现的问题，则是被分摊在了漫长的运维过程中。

很长一段时间，大数据依旧会在各种新名词、易用性(标准SQL、批流、融合、统一)上追逐，运维能力上追逐后端服务。而对于后端服务，则是逐步借鉴大数据的思想，通过K8S这个强大的资源编排系统，将通用的能力落地到资源调度和执行引擎这两层。  
