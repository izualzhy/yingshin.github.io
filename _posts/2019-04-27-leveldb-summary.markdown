---
title: "leveldb笔记之24:leveldb笔记总结"
date: 2019-04-27 18:16:05
tags: leveldb
---

从去年突然想写几篇关于 leveldb 的笔记[开始](https://izualzhy.cn/start-leveldb)，不知不觉已经过去五个月了，原定的几篇也不小心写成了二十多篇。

这篇笔记作为一个总结篇，整体介绍下。

## 1. 整体架构

![architecture-detail](/assets/images/leveldb/architecture-detail.png)

我尝试把所有笔记画成了一张图，包含了重要的结构及流程。从最开始的 leveldb 读写流程逐渐放大，包含 memtable sstable 等的结构，以及更细节的部分。

其实可以画的更细，不过因为太复杂放弃了，感兴趣的话可以直接看下[笔记](https://izualzhy.cn/tags.html#leveldb).

简单来讲，文件结构上跟[BigTable](https://research.google.com/archive/bigtable-osdi06.pdf)里 Tablet Representation 部分很像:

![Tablet Representation](/assets/images/leveldb/leveldb-bigtable.png)

## 2. LSM

提起 leveldb，就不得不说 LSM. leveldb 是 LSM 的一个典型实现。

[The Log-Structured Merge-Tree (LSM-Tree)](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.44.2782&rep=rep1&type=pdf)发表于 1996 年，主题思想就是尽量顺序写磁盘而不是随机写。可能是论文太过久远的关系，这些思想现在都比较普遍了。

更加推荐的是 ben stopford
 的这篇文章：[Log Structured Merge Trees
](http://www.benstopford.com/2015/02/14/log-structured-merge-trees/)，15年发表，年代更近一些，总结的非常全面。

## 3. 总结

提交了一个注释的[leveldb 代码分支](https://github.com/yingshin/leveldb_more_annotation)，结合注释，笔记会更清楚一些。

看到阿里最近的一篇文章：[如何优雅的通过Key与Value分离降低写放大难题？](https://mp.weixin.qq.com/s/ClS1xfQsV7Shx0BcE9GJYg)，已经在解决 LSM 的写放大问题了，顿觉得数据库技术路漫漫其修远兮。我在工作中对 leveldb 使用不多，没有什么实际经验。而大多时候，一些疑难杂症、线上问题，才能真正有助于我们去理解一个系统。因此笔记难免有错误之处也请指出。

