---
title: "浅谈 Flink - State 之 Kafka Offsets"
date: 2020-07-18 19:09:52
tags: [flink-1.9]
---

## 1. 概念

Flink 支持[有状态的计算](https://ci.apache.org/projects/flink/flink-docs-master/concepts/stateful-stream-processing.html)。状态即历史数据，例如计算过程中的 pv uv，此外还有数据源例如 Kafka Connector 的 offsets，数据存储比如接收后缓存未持久化的数据。

计算 uv，就需要存储键为 u，值为 count/明细的数据，可以使用外部存储，也可以在计算引擎中存储。在计算引擎中存储的好处是可以做到对用户无感知，例如`SELECT user, count(distinct url) GROUP BY user`，如果我们只需要写出这样的逻辑，而不用关注`distinct url`是如何存储的，会是一件很美好的事情。当然同样需要解决接口上的易用性、数据不丢不重的可靠性这类基础问题。

Flink 支持这类需求的机制就是 State.

网上介绍 state 的文章很多，都很全面：KeyedState OperatorState，各类 backend 等。我希望从细处着手逐步勾勒出 state 的全貌，通过一系列的笔记记录自己的学习过程，也能讲清楚所以然.

这篇笔记，先从 Kafka Offsets 的基础使用讲起.

## 2. Kafka unionOffsetStates

`FlinkKafkaConsumerBase`使用`ListState`来保存 offsets:

```java
public abstract class FlinkKafkaConsumerBase<T> extends RichParallelSourceFunction<T> implements
        CheckpointListener,
        ResultTypeQueryable<T>,
        CheckpointedFunction {
    ...
    /** Accessor for state in the operator state backend. */
    private transient ListState<Tuple2<KafkaTopicPartition, Long>> unionOffsetStates;
    ...
```

其中`KafkaTopicPartition`定义为:

```java
public final class KafkaTopicPartition implements Serializable {
    ...
    private final String topic;
    private final int partition;
    private final int cachedHash;
```

可以看到这里是很清晰的 Map 结构： topic + partition -> offsets.这些结构要怎么存储，就是 StateBackend 要解决的问题。

当我们使用 Kafka 作为数据源表时，[指定Offsets](https://izualzhy.cn/flink-source-kafka-connector#52-%E6%8C%87%E5%AE%9Aoffsets)的方式(DataStream API也是类似)仅在没有指定 state 的情况下生效。如果启动时指定了 state，则优先从 state 里恢复 offsets，

```java
    public final void initializeState(FunctionInitializationContext context) throws Exception {
        OperatorStateStore stateStore = context.getOperatorStateStore();

        ...

        this.unionOffsetStates = stateStore.getUnionListState(new ListStateDescriptor<>(
                // "topic-partition-offset-states"
                OFFSETS_STATE_NAME,
                TypeInformation.of(new TypeHint<Tuple2<KafkaTopicPartition, Long>>() {})));
```

该函数从 state 读取存储的 offsets，TM 的日志里会输出对应值:

```
Consumer subtask 0 restored state: {KafkaTopicPartition{topic='xxx', partition=0}=xxx}.
```

跟其他 state 一样，从易用性的角度，我们在实际应用时，从某个 checkpoint 目录恢复任务，会希望预先知道对应的 offset 是多少。在 flink 的[State Processor API - Reading State](https://ci.apache.org/projects/flink/flink-docs-master/dev/libs/state_processor_api.html#reading-state)里，提供给了对应的接口。因此下一节介绍下如何读取。

## 3. Reading State

首先我们构造一条数据流：订阅 Kafka 的 json 数据，解析其中的 weight 字段，记录收到的各个 weight 的次数，使用 FsStateBackend 存储 state.

```scala
/**
 * Description: 订阅 Kafka 的 json 数据，解析其中的 weight 字段，记录收到的各个 weight 的次数。
 *
 */
object StreamWithStateSample extends App {
  val params = ParameterTool.fromArgs(args)
  val env = StreamExecutionEnvironment.getExecutionEnvironment
  val fsStateBackend = new FsStateBackend(params.get("ckdir"))
  env.setStateBackend(fsStateBackend)
  env.enableCheckpointing(60000)
  env.getCheckpointConfig.enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION)

  val properties = new Properties()
  properties.setProperty("bootstrap.servers", s"${params.get("brokers")}")
  val consumer = new FlinkKafkaConsumer011(
    params.get("source_topic"),
    new JSONKeyValueDeserializationSchema(false),
    properties
  )

  // 订阅 kafka 数据， 使用 CountFunction 保留并计算其总数
  env.addSource(consumer)
    .uid("source_uid")
    .map(_.get("value").get("weight").asInt())
    .keyBy(i => i)
    .map(new CountFunction)
    .uid("count_uid")
    .print()

  env.execute()

  class CountFunction extends RichMapFunction[Int, (Int, Long)] {
    var countState: ValueState[Long] = _

    override def open(parameters: Configuration): Unit = {
      val countStateDesc = new ValueStateDescriptor[Long]("count", createTypeInformation[Long])
      countStateDesc.setQueryable("count_query_uid")
      countState = getRuntimeContext.getState(countStateDesc)
    }

    override def map(value: Int): (Int, Long) = {
      val currentCount = countState.value()
      val newCount = if (currentCount != null) currentCount + 1 else 1L

      countState.update(newCount)
      (value, newCount)
    }
  }
}
```

我们分别使用 "source_uid" "count_uid" 用于后续获取 KafkaOffsets 以及 count 的 State.
*注： countStateDesc.setQueryable 用于实时读取，当前不用关注*

程序运行后，推送一些数据到指定的 topic：

```
{"weight": 1}
{"weight": 2}
{"weight": 3}
{"weight": 2}
{"weight": 3}
{"weight": 3}
{"weight": 4}
{"weight": 5}
{"weight": 6}
{"weight": 7}
{"weight": 8}
```

TM 输出:

```
1> (4,1)
1> (6,1)
1> (8,1)

2> (1,1)
2> (3,1)
2> (3,2)
2> (2,1)
2> (2,2)
2> (3,3)
2> (5,1)
2> (7,1)
```

当 checkpoint 成功后，`(1,1) (2,2) (3,3)`这些 (key, count) 的数据就写入了对应的 backend.我们看下如何读取:

```scala
package cn.izualzhy.flink

import org.apache.flink.api.common.state.{ValueState, ValueStateDescriptor}
import org.apache.flink.api.common.typeinfo.{TypeHint, TypeInformation}
import org.apache.flink.api.java.ExecutionEnvironment
import org.apache.flink.api.java.tuple.Tuple2
import org.apache.flink.api.java.utils.ParameterTool
import org.apache.flink.configuration.Configuration
import org.apache.flink.runtime.state.memory.MemoryStateBackend
import org.apache.flink.state.api.Savepoint
import org.apache.flink.state.api.functions.KeyedStateReaderFunction
import org.apache.flink.streaming.api.scala.createTypeInformation
import org.apache.flink.streaming.connectors.kafka.internals.KafkaTopicPartition
import org.apache.flink.util.Collector

/**
 * Description: 读取 StreamWithStateSample 写入的 State 数据.
 *
 */
object ReadSampleState extends App {
  val bEnv = ExecutionEnvironment.getExecutionEnvironment
  val params = ParameterTool.fromArgs(args)

  val restoredPath = params.get("ckdir")
  val existingSavepoint = Savepoint.load(
    bEnv,
    restoredPath,
    new MemoryStateBackend())

  val sampleCountStates = existingSavepoint.readKeyedState("count_uid", new SampleStateReaderFunction)
  sampleCountStates.printOnTaskManager("count_uid")
  val kafkaOffsetsState = existingSavepoint.readUnionState(
    "source_uid",
    "topic-partition-offset-states",
    TypeInformation.of(new TypeHint[Tuple2[KafkaTopicPartition, java.lang.Long]]() {}))
  kafkaOffsetsState.printOnTaskManager("source_uid")

  bEnv.execute()

  class SampleStateReaderFunction extends KeyedStateReaderFunction[java.lang.Integer, (Int, Long)] {
    var countState: ValueState[Long] = _

    override def open(parameters: Configuration): Unit = {
      val countStateDesc = new ValueStateDescriptor[Long]("count", createTypeInformation[Long])
      countState = getRuntimeContext.getState(countStateDesc)
    }

    override def readKey(key: java.lang.Integer, ctx: KeyedStateReaderFunction.Context, out: Collector[(Int, Long)]): Unit = {
      out.collect((key, countState.value()))
    }
  }
}
```

`readKeyedState` `readUnionState`返回的是一个`DataSet`，因此用户仍然可以以习惯的方式进行处理，因为我们也可以把 state 看成是一份"数据"。

接口上使用是比较简单的，不过注意这里也会有一些小坑，例如 key 的类型需要是`java.lang.Integer`而不是 Scala 里的`Int`，flink 的 scala 接口并不友好，这个跟其核心部分的实现都是 java 同时兼容 scala 的功能不全有关。

通过上述例子可以正常读取出 state 值：
```
count_uid> (1,1)
count_uid> (3,3)
source_uid> (KafkaTopicPartition{topic='TestTopic', partition=0},6)
source_uid> (KafkaTopicPartition{topic='TestTopic', partition=1},6)
count_uid> (2,2)
```

这里其实只是一个 demo，当我们想继续深入使用这个功能时，就不得不再去思考几个疑问：

1. 读取 State 强依赖[Assigning Operator IDs
](https://ci.apache.org/projects/flink/flink-docs-master/ops/state/savepoints.html#assigning-operator-ids)，当我们使用 flink-SQL 时，是否可以指定 uid，生产环境想要读取 checkpoint 的 offsets 做 DoubleCheck 应该怎么做？  
2. 修改并发为什么会影响到 State？  
3. 为什么会有 KeyedState、OperatorState，为什么会有多种 backend 的需求？

## 4. Ref
1. [State Processor API](https://ci.apache.org/projects/flink/flink-docs-master/dev/libs/state_processor_api.html)  
2. [ScalaWay: Scala学习笔记](https://github.com/yingshin/ScalaWay/tree/master/flink)  
