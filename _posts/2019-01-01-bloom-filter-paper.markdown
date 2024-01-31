---
title: "papers of bloom filter"
date: 2019-01-01 17:07:19
tags: bloomfilter
---

bloom filter 的相关论文较多，按照时间线整理了下自己觉得比较有意义的。有些观点没有找到佐证的中文资料，如有错误还请指出。

## 1. [Space/Time Trade-offs in Hash Coding with Allowable Errors](http://dmod.eu/deca/ft_gateway.cfm.pdf)

Burton Bloom 在1970年发表的文章，布隆过滤器的开山之作。

给定一个 set，查找某个 key 是否存在于该 set，通常考虑两点：

1. time: 查找时间  
2. space: 空间成本(例如 hash 目标区域大小)  

Burton Bloom 在论文里提出了第三点：Allowable Fraction of Errors. ，即允许一定概率的误判，来获取空间成本的显著降低。

**A Sample Application** 一节举了一个 auto hyphenation 的例子，90%的英文单词 hyphenate(过去分词?)都可以通过简单的规则推导出来，剩余10%的单词有一个词典的映射关系，这个关系表比较大，因此只能存储到硬盘。

为这10%的单词建立一个普通的 hash table 的话，内存占用巨大。对应的解决办法，是建立一个 bloom filter，其空间占用很小。正常情况下，先在 bloom filter 查找 key 是否存在，如果不存在，则直接应用简单规则，如果存在，那么查找词典的映射关系。当然，存在一定的误报率，此时浪费的是多查找一次磁盘，只要概率较低，是可以接受的。

![standard bloom filter](assets/images/standard_bloom_filter.png)

## 2. [Summary Cache: A Scalable Wide-Area Web Cache Sharing Protocol](http://pages.cs.wisc.edu/~jussara/papers/00ton.pdf)

这篇论文使得 bloom filter 真正意义上流行开来，[leveldb笔记之9:bloom filter
](https://izualzhy.cn/leveldb-bloom-filter)的理论知识部分大部分出自该论文。

作者提出了 Web Cache Sharing Protocol，与已经存在的**ICP**(Internet Cache Protocol)对比，能有效的降低超过 50% 的带宽，大概 30%~95% 的 protocol cpu消耗，同时保持了跟 ICP 相同的缓存命中率。

假设面对一个海量数据的 cache 问题：我们有 N 个 proxy，每个都拥有自己的 cache，proxy 之间 cache-key 互不相同，当缓存未命中当前 proxy 时，如何知道其他 proxy 是否有对应的 cache 存在？

假设有 N 个 proxy，cache 命中率为 H，每个 proxy 的 qps 为 R.

ICP 的解决方案是发送给其他所有 proxy，那么每个 proxy 需要处理其他 proxy 发送来的 ICP 请求量为

```
(N - 1) * (1 - H) * R
```

整体 proxy 为

```
N * (N - 1) * (1 - H) * R
```

这无疑增加了网络拥塞。

而 Summary Cache，每个 proxy 将自己的 URLs 映射到 BloomFilter，不同 proxy 之间定期交换对方的 BloomFilter，当发生 cache miss 时，当前 proxy 只会发给 BloomFilter 显示存在的对应其他 proxy.

论文提出了两种异常情况：

1. False positive: ProxyA 认为 ProxyB 缓存了 urlU，于是访问 ProxyB 获取对应的数据，实际上 ProxyB 没有，因此返回给 ProxyA 空。ProxyA 访问外网获取数据。这种情况即上篇笔记里的 false positive，会导致多一次 Proxy 的交互。  
2. False negative: ProxyA 认为其他 Proxy 都没有缓存 urlU，于是直接访问外网获取数据。这会导致多一次外网请求。  

这两种异常情况只要控制在较小的概率，是可以接受的。

普通的 BloomFilter 结构只支持添加和查找 key，因此适用于静态的数据。如果 keys 是动态变化的，那么就需要支持删除的功能，论文提出了 Counting Bloom Filter，主要思想是将 BloomFilter 的 bit 扩展为一个小的计数器(Counter)，作者证明了 Counter 最大值可以为4，超过最大值的几率已经非常小了.

![counting bloom filter](/assets/images/counting_bloom_filter.png)

## 3. [Network Applications of Bloom Filters: A Survey](https://www.eecs.harvard.edu/~michaelm/postscripts/im2005b.pdf)

这篇文章我读起来感觉更像是对 BloomFilter 诞生以来的一个总结：更加细致的证明，更多丰富的网络应用例子：Distributed Caching;P2P/Overlay Networks; Resource Routing; Packet Routing; Measurement Infrastructure.

>The Bloom filter principle: Whenever a list or set is used, and space is consideration, a
Bloom filter should be considered. When using a Bloom filter,
consider the potential effects of false positives.”

我的感觉是，实际工程项目里，Bloom Filter 也已经越来越广泛了，例如 [Chromium](https://cs.chromium.org/chromium/src/components/rappor/bloom_filter.cc?sq=package:chromium&g=0)，[squid cache](https://wiki.squid-cache.org/SquidFaq/CacheDigests), leveldb 等。

## 4. 参考

1. [wiki](https://en.wikipedia.org/wiki/Bloom_filter#Counting_filters)
2. [Three papers on Bloom filters](http://bryanpendleton.blogspot.com/2011/12/three-papers-on-bloom-filters.html)
3. [BLOOM FILTERS & THEIR APPLICATIONS](https://pdfs.semanticscholar.org/d899/05bdf1ff791bdddc7c471070f34f4da18844.pdf)

