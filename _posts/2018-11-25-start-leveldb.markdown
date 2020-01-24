---
title: "leveldb笔记开篇"
date: 2018-11-25 16:56:29
excerpt: "leveldb笔记开篇"
tags: [leveldb]
---

leveldb是个高性能、可靠的单机Key-Value数据库，其软件架构设计以及代码实现都非常值得学习。

数据库的实现可以很简单，例如

```cpp
std::map<std::string, std::string> KV;
```

这应该是一个极简版的单机KV数据库了，支持`Put/Delete/Get`操作，数据全部存储在内存。

使用内存存储数据存在两个问题：

1. 数据在进程重启后就会丢失  
2. 内存大小优先，因此数据存储的KV pair数目有限  

为了保证可靠性以及获取更高的容量，数据持久化到磁盘是一个通用的解决办法。

因此我们的数据库实现可以进一步优化为：

```cpp
class KV {
    std::map<std::string, std::string> KV;
    int fd;

    bool Put(key, value) {
        //首先确保数据持久化，然后再写入内存，之后返回用户写入成功
        fd.write(key, value);
        KV.insert(key, value);

        return true;
    }
};//KV

```

`fd`写入数据要考虑这两个问题：

1. 对写入来讲，磁盘的顺序写性能要远远高于随机写，为了高可靠，写入操作需要先写入磁盘再更新内存，因此这里应该是顺序写以尽快返回。  
2. 对读取来讲，如果数据都是顺序写入磁盘，所以`key`是无序的。那么每次读取都要遍历所有文件，读性能会非常差。  

所以，一个高性能的单机数据库，往往是如何平衡读性能、写性能、存储大小的问题，思考这个问题，才能够理解 leveldb 的`write-ahead logging` `MemTable` `SSTable`等设计。leveldb 正是通过这一系列架构设计和代码技巧，提供了一个优秀的存储引擎解决这个问题。

接触 leveldb 已经很长时间了，最近忍不住想要全面梳理一遍。因为发现现在越来越多的存储直接依赖了 leveldb 或者其设计思想，而 leveldb 的实现，从架构到细节确实也有很多借鉴之处。

网上也有众多介绍 leveldb 的文章，相比其他，希望自己梳理的这一系列笔记，能够更容易的让读者深入理解 leveldb：

1. 代码的注释我提交到了[leveldb_more_annotation](https://github.com/yingshin/leveldb_more_annotation)，结合笔记和代码注释，更容易深入理解 leveldb.  
2. 化整为零，通过类级别的测试和说明文档，聚焦到单个类的功能，例如直接生成 SSTable file，观察其存储格式。  
3. 同时介绍软件架构及实现，避免因为沉浸在实现细节而不理解架构设计目的；或者只知道设计目的而忽略了动手实现的能力。  
