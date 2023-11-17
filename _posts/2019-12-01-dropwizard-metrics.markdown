---
title: "监控系统之度量系统：Dropwizard Metrics"
date: 2019-12-14 21:21:24
tags: mixed
---

[Dropwizard Metrics Library](http://metrics.dropwizard.io/) 是一个 java 的监控工具包，Spark 将其作为 monitor 系统的基础实现。借助 Dropwizard Metrics 我们可以通过仅仅几行代码，就可以实现诸如数据分布、延时统计、计数等统计需求，将内部状态暴露出来。对应的，Metrics 实际上包含了两部分，监控的指标(Metric)以及指标如何导出(Reporter)。

## 1. Metric

### 1.1. Meter

Meter 是一个与时间有关的统计值，例如我们可以这么使用：

```scala
  val requests = metrics.meter("requests")

  new Thread {
    ...
    requests.mark()
  }

```

底层则会自动生成 5 个变量值 count(总量)、mean rate(平均值)、1-minute rate(1分钟内平均值)、5-minute(5分钟内平均值)、15-minute(15分钟内平均值)

很多人应该会注意到这个跟 linux 下 load average 的统计很像，实际上底层使用的就是一套算法。Meter 里`ExponentialMovingAverages`负责该功能，其中算法部分交给`EWMA`计算，后者就是按照[Exponential_moving_average](https://en.wikipedia.org/wiki/Moving_average#Exponential_moving_average)实现的。

linux 的 load average 实现可以参考这两篇文章：
1. https://www.helpsystems.com/resources/guides/unix-load-average-part-1-how-it-works  
2. https://www.helpsystems.com/resources/guides/unix-load-average-part-2-not-your-average-average  

其中第二篇提到了这个公式：

```
load(t) = load(t - 1) e-5/60 + n(1 - e-5/60) (1)
```

在[EWMA.java](https://github.com/dropwizard/metrics/blob/4.1-development/metrics-core/src/main/java/com/codahale/metrics/EWMA.java)里可以看到其对应的实现部分：

```scala
    private static final int INTERVAL = 5;
    private static final double SECONDS_PER_MINUTE = 60.0;
    private static final int ONE_MINUTE = 1;
    private static final int FIVE_MINUTES = 5;
    private static final int FIFTEEN_MINUTES = 15;
    private static final double M1_ALPHA = 1 - exp(-INTERVAL / SECONDS_PER_MINUTE / ONE_MINUTE);
    private static final double M5_ALPHA = 1 - exp(-INTERVAL / SECONDS_PER_MINUTE / FIVE_MINUTES);
    private static final double M15_ALPHA = 1 - exp(-INTERVAL / SECONDS_PER_MINUTE / FIFTEEN_MINUTES);

    ...

    /**
     * Mark the passage of time and decay the current rate accordingly.
     */
    public void tick() {
        final long count = uncounted.sumThenReset();
        final double instantRate = count / interval;
        if (initialized) {
            final double oldRate = this.rate;
            rate = oldRate + (alpha * (instantRate - oldRate));
        } else {
            rate = instantRate;
            initialized = true;
        }
    }
```

通过 reporter 可以获取到这些值：

```
-- Meters ----------------------------------------------------------------------
requests
             count = 1
         mean rate = 1.02 events/second
     1-minute rate = 0.00 events/second
     5-minute rate = 0.00 events/second
    15-minute rate = 0.00 events/second
```

当然 linux 源码里实现时要更复杂一些：<https://github.com/torvalds/linux/blob/master/kernel/sched/loadavg.c>，而在 Metrics 里则更追求功能的分装，例如`Meter`持有的是基类对象`MovingAverages`

![MovingAverages](/assets/images/spark/MovingAverages.jpg)

`SlidingTimeWindowMovingAverages` 内部则通过持有 buckets 数组的方式来实现，数组大小为 15min * 60s/1s，每秒的数据都会记录下来并且根据传入的时间参数(1m/5m/15m)来遍历对应的额 buckets 计算，这种简单直接的实现方式可能会更加符合我们最初的理解。

### 1.2. Gauges

Guages 用于提供一个数据量的值，例如我们构造了一个 Gauges 对象用于观测队列 l 的大小

```scala
//  Gauge
  val l = mutable.MutableList[Int]()
  metrics.register(
    MetricRegistry.name(l.getClass, "l", "size"),
    new Gauge[Int]() {
      override def getValue: Int = l.size
    }
  )
```

对象自动绑定到 l，通过 report 可以获取 l 大小

```
-- Gauges ----------------------------------------------------------------------
scala.collection.mutable.MutableList.l.size
             value = 1
```

Gauges 也衍生除了[RatioGuages CachedGuages JMXGuages](https://metrics.dropwizard.io/4.1.1/manual/core.html#gauges)等进一步的度量值记录方式，

### 1.3. Counter

Guages 的优点是可以直接绑定到容器(queue、List等)上，但对应的也限制了其使用的范围。有的时候我们只是希望计数，或者其值并不能简单通过容器来实现`getValue`方法。

这个时候就需要 Counter 登场了。

Counter 是 Guages[AtomicLong] 的一个特化，但是使用更加灵活，例如：

```scala
//  Counter
  val pendingJobs = metrics.counter(
    MetricRegistry.name("myclass", "pending-jobs"))

...
      if (r == 0) {
        pendingJobs.dec()
      } else {
        pendingJobs.inc()
      }
```

暴露出来则是

```
-- Counters --------------------------------------------------------------------
myclass.pending-jobs
             count = 1
```

### 1.4. Histograms

Histograms 用于提供一个数据的分布值，简单讲就是数据的分位值。实时系统的指标，通过保留全部数据然后 sort 明显是不可行的，之前在 brpc 里见过一个[percentile](https://github.com/apache/incubator-brpc/blob/master/src/bvar/detail/percentile.h)，看大意是通过保证概率相等的情况下，仅保留一定量的数据。

其结构跟 Meter 类似，实现上由`private final Reservoir reservoir`完成，该类为虚基类，有多种实现。

![Reservoir](/assets/images/spark/Reservoir.jpg)

官方介绍参考[Histograms](https://metrics.dropwizard.io/4.1.1/manual/core.html#histograms)

#### 1.4.1. UniformReservoir

内部为`DEFAULT_SIZE = 1028`的一个数组

超过大小了则随机丢弃或者不加入：

```scala
    public void update(long value) {
        final long c = count.incrementAndGet();
        if (c <= values.length()) {
            values.set((int) c - 1, value);
        } else {
            final long r = ThreadLocalRandom.current().nextLong(c);
            if (r < values.length()) {
                values.set((int) r, value);
            }
        }
    }
```

来源于这篇论文：[Vitter's R](http://www.cs.umd.edu/~samir/498/vitter.pdf)

#### 1.4.2. SlidingWindowReservoir

保留最近的 N 个值，N 可以在构造函数指定

```scala
    public synchronized void update(long value) {
        measurements[(int) (count++ % measurements.length)] = value;
    }

```

#### 1.4.3. SlidingTimeWindowReservoir

保留最近 N 秒的数据，N 可以在构造函数指定

```scala
    public void update(long value) {
        if (count.incrementAndGet() % TRIM_THRESHOLD == 0) {
            trim();
        }
        measurements.put(getTick(), value);
    }
```

缺点是如果瞬间流量很大，该数据结构内存不可控，因此还提供了`SlidingTimeWindowArrayReservoir`这个替代的基础结构。

#### 1.4.4. ExponentiallyDecayingReservoir

`MetericRegistry::histogram`默认创建为为该类型，内部为`DEFAULT_SIZE = 1028`的一个跳表

```
    public void update(long value, long timestamp) {
        rescaleIfNeeded();
        lockForRegularUsage();
        try {
            final double itemWeight = weight(timestamp - startTime);
            final WeightedSample sample = new WeightedSample(value, itemWeight);
            final double priority = itemWeight / ThreadLocalRandom.current().nextDouble();

            final long newCount = count.incrementAndGet();
            if (newCount <= size || values.isEmpty()) {
                values.put(priority, sample);
            } else {
                Double first = values.firstKey();
                if (first < priority && values.putIfAbsent(priority, sample) == null) {
                    // ensure we always remove an item
                    while (values.remove(first) == null) {
                        first = values.firstKey();
                    }
                }
            }
        } finally {
            unlockForRegularUsage();
        }
    }
```

来源于这篇论文：[forward-decaying priority reservoir](http://dimacs.rutgers.edu/~graham/pubs/papers/fwddecay.pdf)

### 1.5. Timer

Timer 用于时间统计，例如一个接口的相应时间，相当于 Meter + Histograms.

```scala
    private final Meter meter;
    private final Histogram histogram;
```

使用时可以借助其 time 方法，不过在 Scala 里没有找到类似 C++ RAII 的资源管理方式，因此实现上比较 trick(貌似2.13后可以使用[Using](https://scala-lang.org/files/archive/api/2.13.0/scala/util/Using$.html))

```scala
//  Timer
  val responseTimer = metrics.timer(
    MetricRegistry.name("myclass", "response-timer")
  )
...
  def foo(): Unit = {
    Thread.sleep(500 + scala.util.Random.nextInt(500))
  }

//  def using[T <: AutoCloseable, M](resource: T)(block: (M) => Unit)(args: M): Unit = {
  def using[T <: AutoCloseable, M](resource: T)(block: () => Unit): Unit = {
//    def using[T <: { def close() }, M](resource: T)(block: (M) => Unit)(args: M): Unit = {
    try {
      block()
    } finally {
      if (resource != null) resource.close()
    }
  }
...
      using(responseTimer.time())(foo)

```

## 2. Reporter

Reporter 支持 JMX/HTTP 等[Reporter](https://metrics.dropwizard.io/4.1.1/getting-started.html#other-reporting)，Spark 环境下则对应到多种的[sink 实现](http://spark.apache.org/docs/latest/monitoring.html)

`MetricRegistry`用于连接各`Metric`和`Reporter`，写了一个简单的使用例子[MetricsSample](https://github.com/yingshin/ScalaLearning/blob/master/ScalaRecipe/src/main/scala/recipe/MetricsSample.scala)

PS：如果有像我一样从 C++ 转过来的 Scala 新手的话，会发现 Metrics 部分跟 brpc 里的 bvar 很像，而 Metrics 里广泛使用的`LongAdder`则同样是避免线程竞争，以空间换时间的处理方式，适合于写多读少的监控场景。

参考：
https://metrics.dropwizard.io/4.1.1/index.html


