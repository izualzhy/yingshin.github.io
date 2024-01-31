---
title: zookeeper介绍
date: 2016-9-24 23:05:08
excerpt: "zookeeper介绍"
tags: zookeeper
---

本文主要介绍zookeeper的数据模型以及常用的四字命令。

<!--more-->

### 1. 数据模型

如果我们自己要实现分布式锁的机制，需要实现三个最重要的锁原语：`create acquire release`。

zookeeper对外提供的是一个树形的数据结构，每个节点就被称为znode。同时zookeeper没有暴露分布式锁的原语，而是提供了一系列的API，这些API操作的对象就是znode。

举个具体的数据树结构的例子：

![znode.png](/assets/images/znode.png)

其中`master tasks slave0 slave1`都是我们创建的znode，每个znode存储了kv数据。更具体的：

1. 树形的数据结构引入了父子节点关系，例如`tasks`有两个子节点`slave0 slave1`，`slave0`的绝对路径为`/tasks/slave0`，所有节点都在root节点`/`下。  
2. znode使用路径来标示key，例如`/master /tasks /tasks/slave0 /tasks/slave1`，创建znode时的路径表示了对应的父子节点关系  
3. znode的value可以设置为一个空字符串""  
4. 对于一个znode，我们可以设置监视其创建、更新、删除、子节点变化等一系列的修改。  
5. znode可以设置操作权限，zookeeper使用ACL来控制。

### 2. zonde的不同类型

当创建znode时，可以指定该节点的类型，不同的类型决定了znode节点的行为方式。

#### 2.1. 持久节点和临时节点

znode节点可以使持久(persistent)节点，还可以是临时(ephemeral)节点。两者的区别主要在删除行为上，持久节点只能通过`delete`删除。而临时节点，当创建了该节点的客户端崩溃或者关闭了与zookeeper服务端的连接时，就会自动删除。

注意ephemeral znode不能有子节点。

#### 2.2 有序节点

有序（sequential)节点创建时，被默认分配一个单调递增的整数，"%10d"的形式添加到给定的路径之后。
例如创建一个有序的znode节点，给定路径为/tasks/tasks-，实际生成的节点路径可能为`/tasks/tasks-0000000000`

```
[zk: 127.0.0.1:2181(CONNECTED) 8] create -s /tasks/tasks- ""
Created /tasks/tasks-0000000000
```

上述来自zk提供的shell，`zkCli`，将在[zkCli的使用](http://izualzhy.cn/zkcli-introduction)单独介绍。

因此znode一共有四种类型：

1. persistent  
2. ephemeral  
3. persistent_sequential  
4. ephemeral_sequential  

### 3. 监视和通知

关于通知一个常用的想法是轮询，客户端不断的查询节点是否znode状态有变化。然而轮询方式并非高效的方式，尤其在期望的znode变化频率很低时。zookeeper使用了一种观察者（observer）的方式来替换客户端的轮询。

这是一种基于通知（notification）的机制：客户端向ZooKeeper注册需要接收通知的znode，通过对znode设置监视点（watch）来接收通知。比如我们注册了`/master`的监视点，当`/master`发生变化时，客户端就会收到一个通知，并从ZooKeeper服务端读取一个新值。注意通知机制是单次触发的操作，如果需要继续监视该znode，需要再次注册监视点。这个单次触发的设计初次接触可能会比较怪，不过在熟悉API以及具体的使用场景后，会发现这个设计使用起来还是比较合理的。

到这里基础介绍算是结束了，接下来的几篇文章，会逐渐进入到zk实战的部分。我们会发现在zookeeper对这些分布式的事务做了良好的封装，为了避免进入到知其然不知其所以然的节奏，在具体的使用zookeeper前，有几个问题先抛出来。我们会逐渐的解释：

1. 单次触发是否会导致丢事件？  
2. 无序的问题是否存在，比如客户端A先创建了nodeA，接着删除，服务端按照这个顺序通知客户端B，但是如果到达顺序为删除nodeA -> 创建nodeA怎么办？  
3. 客户端A B同时去修改一个node的value，结果应该选择哪个？在无法确认网络状况的情况下，如何保证结果符合预期？  
4. 客户端A连接到服务端I，创建了nodeA，断开链接接着重连到服务端II，此时II还未同步到I上创建nodeA的操作，该如何处理？换言之zookeeper是否保证了一致性？  
5. 集群模式下zookeeper允许多少台服务机器出现故障？  

在介绍znode的各种操作前，先介绍下如何查看便捷的查看zookeeper服务端的状态。

### 3. 四字命令

四字命令的主要目标就是提供一个简单的协议，使我们使用telnet/nc工具就可以获取zookeeper服务的当前状态。使用的形式就是`echo ruok | nc 127.0.0.1 2181`。

#### 3.1. ruok

返回服务器运行状态，"imok"表示已经启动

#### 3.2. stat

获取服务器的状态信息，以及当前活动的链接情况。状态信息包括一些基本统计信息，还包括当前服务器是follower/leader。

我们当前的zk为单机模式，因此Mode = standalone.

```
[spider@cp01-saverng-2 bin]$ echo stat | nc 127.0.0.1 2181
Zookeeper version: 3.4.8--1, built on 02/06/2016 03:18 GMT
Clients:
 /127.0.0.1:15542[0](queued=0,recved=1,sent=0)

Latency min/avg/max: 0/0/22
Received: 3520
Sent: 3519
Connections: 1
Outstanding: 0
Zxid: 0x17d
Mode: standalone
Node count: 24
```

#### 3.3 srvr

信息与stat一样，忽略了链接情况的信息

#### 3.4 dump

列出当前活动的会话信息以及对应的过期时间。

#### 3.5 kill

关闭server，不要轻易尝试（囧）

#### 3.6 conf

该服务器启动使用的基本配置参数

```
[spider@cp01-saverng-2 bin]$ echo conf | nc 127.0.0.1 2181
clientPort=2181
dataDir=/home/spider/local/zookeeper-3.4.8/data/version-2
dataLogDir=/home/spider/local/zookeeper-3.4.8/log/version-2
tickTime=2000
maxClientCnxns=60
minSessionTimeout=4000
maxSessionTimeout=40000
serverId=0
```

#### 3.7 cons

列出该服务器所有链接的详细统计信息

```
[spider@cp01-saverng-2 bin]$ echo cons | nc 127.0.0.1 2181
 /127.0.0.1:20095[0](queued=0,recved=1,sent=0)
 /127.0.0.1:17049[1](queued=0,recved=64,sent=64,sid=0x1575ceca89a0006,lop=GETC,est=1474768042776,to=30000,lcxid=0x5,lzxid=0x186,lresp=1474768637650,llat=1,minlat=0,avglat=0,maxlat=2)
```

#### 3.8 envi

服务环境的详细信息（包括java环境)

#### 3.9 wchs

服务器所跟踪的监视点的简短摘要信息

```
[spider@cp01-saverng-2 bin]$ echo wchs | nc 127.0.0.1 2181
2 connections watching 2 paths
Total watches:2
```

#### 3.10 wchc

列出服务器所跟踪的监视点的详细信息，根据会话进行分组。

```
[spider@cp01-saverng-2 bin]$ echo wchc| nc 127.0.0.1 2181
0x1575ceca89a000b
        /ephemerals_node1
0x1575ceca89a000c
        /ephemerals_node2
```

#### 3.11 wchp

列出服务器所跟踪的监视点的详细信息，根据监视点znode路径进行分组

```
[spider@cp01-saverng-2 bin]$ echo wchp| nc 127.0.0.1 2181
/ephemerals_node1
        0x1575ceca89a000b
/ephemerals_node2
        0x1575ceca89a000c
```

### 4. 不适用的场景

zookeeper不适合用于海量数据存储，在hbase中，zookeeper仅仅用于元数据的存储。因为对于一致性和持久性的不同需求，架构设计上应该做到应用数据和协同数据分开。

### 5. 参考资料
1. [ZooKeeper：分布式过程协同技术详解](http://www.duokan.com/book/106575)
