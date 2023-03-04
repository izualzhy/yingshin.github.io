---
title: "DS笔记之2：工作流是如何启动的？"
date: 2023-03-04 15:25:00
tags: [DolphinScheduler-3.1.3]
---

## 1. 何时启动工作流？

对于一个任务调度系统，任务的启动无外乎以下两个入口：
1. 系统调度：例如用户配置的Crontab、上游依赖任务的触发、任务的容错  
2. 手动运行：例如任务测试、补数、重跑失败任务    

这篇笔记主要介绍下，任务生成到数据库后，Master读取、编排DAG、发送到 Worker 的过程。

## 2. 启动流程

DolphinScheduler 官网的这张图概括了整体的流程：![process-start-flow-1.3.0](/assets/images/dolphin/process-start-flow-1.3.0.png)

*注：图片基于V1.3.0，跟当前略有出入*

## 2.1. 生成工作流实例

这个过程是读取 t_ds_command 表，处理后写入 t_ds_process_instance 表，主要两方参与：  
1. `MasterSchedulerBootStrap`: 轮询从 t_ds_command 表选出由且仅由该 master 实例处理的行，存储到`List<Command>`  
2. `masterPrepareExecService`: 线程池，将`List<Command>`转化为`List<ProcessInstance>`  

注意：  
1. 是否 overload 在这一步判断，当前主要是基于 load average 和 mem。实际上 load average 不能直接对标 cpu idle：[扯扯 cpu idle 与 load average](https://izualzhy.cn/sys-idle-load)  
2. Master 通过 ZK 判断当前实例的索引下标，只选取 t_ds_command 表主键取模后跟实例下标一致的行，避免重复调度  
3. 为了提高性能，在 Command -> ProcessInstance 时使用了线程池<sup>[issue](https://github.com/apache/dolphinscheduler/issues/6849)</sup>  

`ProcessInstance`生成后，会封装到`WorkflowExecuteRunnable`，这个类的功能非常庞大，同时封装了后续的构造DAG、任务提交等一系列功能。  
`WorkflowExecuteRunnable`对象会存储到`ProcessInstanceExecCacheManager`，该类内部维护一个`ProcessInstance.id`为 key 的 hash 结构，支持全局查找。id 也会被添加到`WorkflowEventQueue`这个队列，以触发下一步的计算。

## 2.2. 触发工作流实例

`WorkflowEventQueue`、`MasterSchedulerBootstrap`、`WorkflowEventLooper`组成了典型的生产者-消费者模型。

逻辑非常轻量级，就是从队列里取出上一步写入的id，提交`workflowExecuteRunnable::call`到`WorkflowExecuteThreadPool`这个线程池去执行。   

## 2.3. 提交工作流实例

`WorkflowExecuteRunnable::call`真正开始一个工作流。生成DAG、构造TaskInstance对象，都是在这个方法里实现的。运行在`WorkflowExecuteThreadPool`线程池，线程前缀名为"WorkflowExecuteThread-".

### 2.3.1. 构造DAG-buildFlowDag

DAG 的概念很常见，例如 Flink 里的 [JobGrap](https://izualzhy.cn/flink-source-job-graph)。虽然跟 Dolphin 里的 DAG 在需求和实现上都相差巨大，但是两者都有一个共同点，就是要构建**具体运行的算子实例及其关系**。

Process 定义了算子及其依赖关系，而算子的真正执行是在 Task，例如 Shell、SQL、MapReduce、Flink 等任务类型。因此这一步主要是：  
**读取这个 ProcessInstance 下的所有 Task 实现，确定本次都需要启动哪些 Task，其先后顺序是什么。**

Process -> Task 主要有三张表记录：

1. t_ds_process_definition: 数据流定义
2. t_ds_task_definition: 任务定义
3. t_ds_process_task_reltation: 数据流都包含哪些任务

关系如图：
![ProcessTaskMetadata](/assets/images/dolphin/dolphin/metadata-buildflowdag.png)

注意：
1. 任务在开发过程中可能会被不断修改，线上运行中任务、历史任务使用的未必是当前的配置。因此这几张表都会有对应的 \_log 表，保存了历史记录。代码里真正的实现，也都包含了对历史表的查询。
2. `pre_task_code = 0`，则表示该任务没有前置依赖  

`DAG<String, TaskNode, TaskNodeRelation> dag`对象存储了生成后的 DAG，包含需要运行的 Task 节点，以及节点之间的依赖关系。

### 2.3.2. 初始化taskmap-initTaskQueue

### 2.3.3. 提交task-submitPostNode

这一步是提交任务到优先级队列中。首先提交的是 DAG 的开始节点，例如没有前置依赖的节点、直接运行的节点等。前置节点运行结束后，再执行后续节点，这大概也是函数命名`submitPostNode`的由来。

待提交的任务先添加到`readyToSubmitTaskQueue`队列，然后遍历该队列，判断是否满足提交条件。如果满足，则调用`submitTaskExec`方法。

根据任务实例的不同类型，构造对应的 ITaskProcessor 实例，例如 CommonTaskProcessor、DependentTaskProcessor 等。大部分任务类型对应CommonTaskProcessor，会将任务添加到`TaskPriorityQueue<TaskPriority> taskUpdateQueue`优先级队列。

## 2.4. 分发任务实例

该类是一个单线程，也是标准的 Producer-Consumer 模型，从优先级队列选取出任务实例后。调用多线程发送到配置的 Worker 节点。节点的选择、负载均衡都是在这一步完成的。
