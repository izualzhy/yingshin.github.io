---
title: "DolphinScheduler-1: Master、Worker的线程模型"
date: 2023-09-29 12:26:25
tags: dolphin
---

Master、Worker是DolphinScheduler最重要的两个模块，Master负责任务的调度，Worker负责任务的执行。

DolphinScheduler 官网的这张图概括了整体的流程：![process-start-flow-1.3.0](/assets/images/dolphin/process-start-flow-1.3.0.png)

*注：图片基于V1.3.0，跟当前略有出入*

## 1. 线程模型
任务提交过程中，使用到了众多线程，其中最重要的单线程、线程池，如图所示：

![start-process-thread-model](/assets/images/dolphin/dolphin/start-process-thread-model.png)

DolphinScheduler 的线程模型总的来说：  
1. 串联了多个生产者-消费者，队列使用内存队列  
2. 线程定义了独立的名字区分  
3. Master-Worker 之间通过 Netty 通信  

## 2. Master线程模型

定时调度的处理入口类是`class MasterSchedulerBootstrap extends BaseDaemonThread implements AutoCloseable`，这是一个单线程。

1. `findCommands`方法查询出待调度的任务，通过`command2ProcessInstance`转化为`ProcessInstance`。构造`WorkflowExecuteRunnable`，作为生产者传入`WorkflowEventQueue`。   
2. `class WorkflowEventLooper extends BaseDaemonThread`消费`WorkflowEventQueue`，调用`WorkflowStartEventHandler.handleWorkflowEvent`，将`WorkflowExecuteRunnable::call`方法交给`WorkflowExecuteThreadPool`  
3. `class WorkflowExecuteThreadPool extends ThreadPoolTaskExecutor`是一个线程池，执行传入的方法。  
4. `WorkflowExecuteRunnable::call`：根据 ProcessInstance 构造 DAG，即产出多个关联的 TaskInstance，提交到`TaskPriorityQueue`  
5. `class TaskPriorityQueueConsumer extends BaseDaemonThread`单线程消费上述队列，调用`dispatchTask`方法，通过`consumerThreadPoolExecutor`线程池将任务发送到各个Worker节点     

## 3. Worker线程模型

1. Worker接收Master请求的处理类是`class TaskDispatchProcessor implements NettyRequestProcessor`，处理方法`process(Channel channel, Command command)`。构造`WorkerDelayTaskExecuteRunnable`，写入到`DelayQueue<WorkerDelayTaskExecuteRunnable>`  
2. `class WorkerManagerThread implements Runnable`内部启动一个线程，消费上述队列，将任务提交到线程池  
3. `class WorkerExecService`维护一个线程池，执行`WorkerTaskExecuteRunnable::call`方法，该方法会根据具体Task类型执行对应任务：ShellTask/SqlTask/FlinkTask等。