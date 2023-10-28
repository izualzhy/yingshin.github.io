---
title: "知易行难-读《中台落地手记》"
date: 2023-10-21 09:25:21
tags: [read]
---
![中台落地手记](https://izualzhy.cn/assets/images/book/s33990103.jpg)

[中台落地手记 : 业务服务化与数据资产化](https://book.douban.com/subject/35591140/)这本书在豆瓣只有6分，但读起来受益不少。主要有几个原因：

1. 图画的清晰、好看
2. 方便速成
3. 自身做了几年大数据中台

读书笔记按照这几点分别总结。

## 1. 图

一图胜千言。虽然我的 PPT 画的难看，但是我羡慕 PPT 画的清楚的人，也赞同 PPT 文化。PowerPoint 画的清楚，往往代表着思路清晰、重点明确。

图不在于画的好看，而是在于清楚，有取舍、有重点。节取一些架构图：

**大数据平台架构**: ![bigdata-platform](/assets/images/ZhongTaiLuoDiShouJi/bigdata-platform.jpeg)

**数据质量**: ![data-quality](/assets/images/ZhongTaiLuoDiShouJi/data-quality.jpeg) 看这张图的时候，我一直在想数据质量平台应该是先验还是后验平台？

**Spark各阶段**: ![spark-stage](/assets/images/ZhongTaiLuoDiShouJi/spark-stage.jpeg)

**ServiceMesh**: ![service-mesh](/assets/images/ZhongTaiLuoDiShouJi/service-mesh.jpeg)

**MyCat**: ![mycat](/assets/images/ZhongTaiLuoDiShouJi/mycat.jpeg)

**Consul**: ![consul](/assets/images/ZhongTaiLuoDiShouJi/consul.jpeg)

**Apache Atlas**: ![apache atlas](/assets/images/ZhongTaiLuoDiShouJi/apache-atlas.jpeg)

**Apache Griffin**: ![apache atlas](/assets/images/ZhongTaiLuoDiShouJi/apache-griffin.jpeg)

## 2. 速成

读这本书，可以快速的了解：

1. 微服务用 Dubbo 还是 Spring，配置中心、负载均衡、熔断限流、缓存、服务网格、日志监控都用哪些？
2. 大数据技术栈：消息队列、计算引擎、平台、元数据、数据质量、数据湖
3. 数仓：分层、建模、价值判断
4. 运营中台、数据中台、AI中台等等

当然都是蜻蜓点水的介绍，比如消息队列选了 Kafka 而不是 Pulsar，不用考虑太多为什么，能快速搭起来。

这也是我接触的很多大数据组件的共同之处：能解决问题，但不是万金油；上手简单，但出了问题两眼一抹黑-抓瞎🦐

## 3. 感受

中台这个概念，现在慢慢冷一些了。之前看中台，仿佛看到了一个无比强大的中场发动机，前场快速切换、试错。似乎花极少人力，就可以支持公司新产品层出不穷。然而搭建的过程知易行难。

### 3.1. 目标
中台的目标，是需要能够高效支持业务，业务的目标是赚钱。

业务给公司赚钱的。中台，除非是云厂商，都是花钱的，是负债的部门。业务的产品是资产，但是代码、不止中台的代码，都是负债。

代码写出来就要维护，人力变动了要交接，不断的屎上雕花或者重构。用越少的代码支持了产品，负债就越小。

业务为了实现赚钱的目标，需要拉新、投放，各种途径都得先花出去钱。先撒币，才可能赚钱。

中台为了实现高效的目标，需要探索、积累，先允许各种试错才能有沉淀。

这个过程里，业务可能会被砍，中台可能会被质疑、自我怀疑，典型的一些疑问：  
1. A业务当前才几 qps，中台想这么多稳定性的问题干什么？   
2. B业务最近 qps 涨的很快，中台为什么这么多稳定性问题？     
3. 中台的人都在干啥，这个组件、功能我C业务又不需要   
4. 中台偿还技术债务，为什么要业务配合？   
 
因此，目标需要对齐，需求需要管理，中台要敬畏业务。当然建不建中台，也应该是个目标。

### 3.2. 人

其次的问题是人：战略不能心动，使用的人要痛。

只要业务赚钱，数据要不到不是问题，招人可以解决。只有数据太多，头绪太杂，需要快刀斩乱麻了，才会痛。

因此当出现了太多的烟囱架构，使用的痛了，中台的价值就无须费劲证明了。

可是这也是中台最不想看到的，善战者无赫赫之功。这个时候，宣告的是中台的失败。

### 3.3. 事

中台要协调两件事：业务需求和长线积累。

业务需求提过来了，是合并同类项统一规划还是短期上线，短期上线的屁股什么时候擦？

一边做业务需求，一边还之前的技术债务，但技术不往前走，长线积累没有也不行。总不能你还没人力调研 iceberg，业务跳出来说已经要给 iceberg 提代码了。所以中台的事项里，一定都要兼顾。

毛主席也说了，“手里没把米，叫鸡都不来”。
