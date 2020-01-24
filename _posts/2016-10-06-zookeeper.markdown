---
title: zookeeper指南
date: 2016-10-6 00:40:18
excerpt: "zookeeper指南"
tags: [zookeeper]
---

接触zk想起来已经大概有两年多的时间，期间一直没有系统的总结过。最近接手了两个遗留模块，都将zk作为模块的强依赖，但是都存在一些使用上的问题，导致接手后梳理非常困难。因此抽空重新系统的了解了下zookeeper，也是这几篇指南的来历。

<!--more-->

zookeeper的基本功能是在分布式系统中协作多个任务，比如最常见的群首选举的需求，同时在分布式项目中负责配置管理，元数据存储等工作。

具体的，zookeeper一些使用的实例：HBase中用于选举集群内的主节点，以便跟踪可用的服务器，并保存集群的元数据。Kafka中用于检测崩溃，实现topic的发现，并保持topic的生产和消费状态。

zookeeper实现了一组核心操作，通过这些可以实现很多常见分布式应用的任务。例如在一个常见的主从模型中，我们需要实现主节点选举，节点存活与否的跟踪的功能。zookeeper提供了实现这些任务的工具，但是如何选举主节点、协同任务，则由应用层自己决定。

几篇文章主要包括安装、介绍、zkcli、c/python的客户端lib解析及介绍，并推荐阅读[ZooKeeper:Distributed Process Coordination](http://pan.baidu.com/s/1nuT9tUx)以及中文版。

1. [单机zookeeper的安装](http://izualzhy.cn/zookeeper-install)  
2. [zookeeper介绍](http://izualzhy.cn/zookeeper-introduction)  
3. [zookeeper之zkCli的使用](http://izualzhy.cn/zkcli-introduction)  
4. [使用zkcli实现一个主-从模式的架构](http://izualzhy.cn/zkcli-example)  
5. [zookeeper C API介绍](http://izualzhy.cn/zookeeper-c-api-introduction)  
6. [zookeeper-kazoo介绍](http://izualzhy.cn/zookeeper-python-kazoo-introduction)  
7. [c客户端源码分析一：数据结构与线程](http://izualzhy.cn/zookeeper-c-client-src-structure-and-thread)  
8. [c客户端源码分析二：接口调用后发生了什么？](http://izualzhy.cn/zookeeper-c-client-src-user-thread)  
9. [c客户端源码分析三：两个线程](http://izualzhy.cn/zookeeper-c-client-src-lib-thread)  
