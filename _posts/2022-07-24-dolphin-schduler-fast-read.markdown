---
title: "Apache DolphinScheduler 代码速读"
date: 2022-07-24 05:59:58
tags: [dolphin]
---

周末花时间看了下 Apache DolphinScheduler 的代码，先水一篇整理的基础部分笔记占位。

![Arch of DolphinScheduler](/assets/images/dolphin/architecture.jpeg)

系统架构主要分为几各模块：

1. api-server: API接口层，主要负责处理前端UI层的请求。该服务统一提供RESTful api向外部提供请求服务。 接口包括工作流的创建、定义、查询、修改、发布、下线、手工启动、停止、暂停、恢复、从该节点开始执行等等。
2. master-server: MasterServer采用分布式无中心设计理念，MasterServer主要负责 DAG 任务切分、任务提交监控，并同时监听其它MasterServer和WorkerServer的健康状态。 
3. worker-server: WorkerServer也采用分布式无中心设计理念，WorkerServer主要负责任务的执行和提供日志服务。 WorkerServer服务启动时向Zookeeper注册临时节点，并维持心跳。 
4. alert-server: 提供告警相关接口，接口主要包括告警两种类型的告警数据的存储、查询和通知功能。

此外还有 log-server

这几个模块可以在[start-all.sh](https://github.com/apache/dolphinscheduler/blob/dev/script/start-all.sh)找到启动过程。

## 1. MasterServer

`MasterServer`位于`org.apache.dolphinscheduler.server.master`.

`run`方法里主要做了：

1. 启动`NettyRemotingServer`
2. 通过`MasterRegistryClient`注册ZK
3. 调用`QuartzExecutors`启动定时服务


## 2. WorkerServer

`WorkerServer`位于`org.apache.dolphinscheduler.server.worker`.

`run`方法里主要做了：

1. 启动`NettyRemotingServer`
2. 通过`WorkerRegistryClient`注册ZK
3. 启动`WorkerManagerThread`，该类包含了单个任务的具体执行线程类`TaskExecuteThread`
4. 启动`RetryReportTaskStatusThread`