---
title: "DolphinScheduler-8: 状态"
date: 2023-10-04 05:52:30
tags: dolphin
---

在[工作流的生命周期](https://izualzhy.cn/ds-process-lifecycle)里，初步介绍过工作流的各种状态。

## 1. 状态机

如果我们实现一个任务调度系统，首要是梳理清楚任务状态。

以 Flink 任务流程为例：

1. 提交：任务提交后，资源调度需要分配资源、初始化Container、启动JobManager、TaskManager等。因此任务首先是从**初始化**到**提交中**，再从**提交成功**到**运行**，当然任务也有可能因为各种原因导致**提交失败**。   
2. 运行：运行中的任务，可能会**成功**，可能会**失败**，对流式任务，也有可能一直是**运行**。   
3. 停止：任务在任何一个状态，都可能会收到**停止的事件**，同理会先变成**停止中**，再变成**已停止**，如果停止失败，那回到原来的状态。    

![FlinkTaskState](/assets/images/dolphin/dolphin/flink-task-state.png)

这个流程包含三元素：  
1. 状态：初始化、提交中、运行、成功、失败、已停止，都是任务的状态   
2. 事件：提交任务、停止任务时，都会触发对应的事件(任务提交、任务停止)   
3. 动作：响应事件，执行某个方法，然后任务切换到另一个状态   

简单讲，就是触发不同的事件(主动/被动)后，执行某个动作，使得状态变了。  

假如依次收到三个事件：`提交任务 -> 停止任务 -> 提交任务`   
如果执行顺序弄成了：`提交任务 -> 提交任务 -> 停止任务`     

那任务的最终状态就不符合预期了，因此**事件必须顺序处理**。   

另外从实现的角度，有的过程是无法打断的，比如提交任务，强行 interrupt 提交线程结果是未知的，结果任务可能提交了，也可能没有提交。最保险的做法，是等提交完成后再去停止任务。

因此可以得到结论：**事件是多线程触发的，但是为了确保顺序性，动作应该单线程执行**   

上面的描述很容易让人联想到状态机，不过状态机实际是一个抽象的概念，更多是一种约定而非限制。比如[Raft](https://izualzhy.cn/notes-on-raft)协议里的 Replicated state machines、资源管理系统(YARN)里的任务状态：NEW_SAVING ACCEPTED RUNNING 等，都是状态机的一种实现。  

就像设计模式一样，没有固定成法，但是如果提前知晓模块使用哪种模式实现，读代码就会顺畅很多。  

这种就是**最通用的编程语言**，因此在这篇笔记里扯扯状态机。

## 2. DolphinScheduler 的状态机

上一节介绍的是单个 Flink 任务的状态，Dolphin 里要复杂一些，原因在于：   
1. 工作流是由多个任务组成的：任务完成后，需要触发 DAG 的下一个任务执行   
2. 任务状态不等于工作流状态：工作流的状态需要根据 DAG 里多个任务综合判断    

不过说到底还是一样的，典型的实现方式：   

![StateMachine](/assets/images/dolphin/dolphin/state-machine.png)

事件放到队列，通过队列依次处理可以**确保顺序性**。

由于任务之间的事件是互不影响的，因此为了提高性能，可以只**将同一个任务的事件顺序处理**。   

处理方法可能叫做 handle、process、action、operate，只是不同场景命名的区别。    
DolphinScheduler 无论对于 Task 还是 Process 的状态变更，本质都是这套方式。两个状态独立变更，代码实现分别在`TaskEventService`、`EventExecuteService`    

## 3. TaskEventService

`TaskEventService`负责任务状态的变化。

![TaskEventModel](/assets/images/dolphin/dolphin/task-event-model.png)

从左到右依次是：  
1. `TaskEvent`: 事件描述，例如任务分发、任务完成、任务拒绝等       
2. `TaskEventService`: 事件的分发，需要同时考虑顺序性和性能   
3. `TaskEventHandler`: 事件处理，每种类型的事件，都会有对应的处理方法     

之前介绍了工作流的启动过程，DAG 里的首节点任务开始执行；而当首节点任务结束后，就需要触发 DAG 里下游节点开始执行。这便是状态机的作用之一，接下来介绍下这个该过程。    

### 3.1. 事件产生   

worker 执行完成任务后，发送`TASK_EXECUTE_RESULT`事件回 master

```java
public abstract class WorkerTaskExecuteRunnable implements Runnable {
	...
    protected void sendTaskResult() {
        ...
        workerMessageSender.sendMessageWithRetry(taskExecutionContext, masterAddress, CommandType.TASK_EXECUTE_RESULT);

        logger.info("Send task execute result to master, the current task status: {}",
                taskExecutionContext.getCurrentExecutionStatus());
    }
```

master 调用对应的 NettyRequestProcessor 处理消息，这里即`TaskExecuteResponseProcessor`:

```java
public class TaskExecuteResponseProcessor implements NettyRequestProcessor {
	...
    @Override
    public void process(Channel channel, Command command) {
        Preconditions.checkArgument(CommandType.TASK_EXECUTE_RESULT == command.getType(),
                                    String.format("invalid command type : %s", command.getType()));

        TaskExecuteResultCommand taskExecuteResultMessage = JSONUtils.parseObject(command.getBody(),
                                                                                  TaskExecuteResultCommand.class);
        TaskEvent taskResultEvent = TaskEvent.newResultEvent(taskExecuteResultMessage,
                                                             channel,
                                                             taskExecuteResultMessage.getMessageSenderAddress());
        try {
            LoggerUtils.setWorkflowAndTaskInstanceIDMDC(taskResultEvent.getProcessInstanceId(),
                                                        taskResultEvent.getTaskInstanceId());
            logger.info("Received task execute result, event: {}", taskResultEvent);

            taskEventService.addEvent(taskResultEvent);
```

可以看到`TaskEvent`对象放到了 taskEventService 里。

注：RPC的过程参考上篇笔记[DolphinScheduler-7: 网络模型](https://izualzhy.cn/ds-net-model)

### 3.2. 事件分发   

`TaskEventService`的代码不多，主要有两个线程、一个线程池组成：

1. `TaskEventDispatchThread`取出 taskEvent，放到`ConcurrentHashMap<Integer, TaskExecuteRunnable> taskExecuteThreadMap`，其中`TaskExecuteRunnable`保存了该工作流实例下的所有 taskEvent    
2.`TaskEventHandlerThread`遍历`taskExecuteThreadMap`，将`TaskExecuteRunnable.run`提交到线程池执行。定义了`multiThreadFilterMap`以确保同一个工作流下任务事件的顺序性         

注意真正执行都是在`TaskExecuteThreadPool`线程池里。

### 3.3. 事件处理   

任务事件处理的基类：

```java
public interface TaskEventHandler {

    /**
     * Handle the task event
     *
     * @throws TaskEventHandleError     this exception means we will discord this event.
     * @throws TaskEventHandleException this exception means we need to retry this event
     */
    void handleTaskEvent(TaskEvent taskEvent) throws TaskEventHandleError, TaskEventHandleException;

    TaskEventType getHandleEventType();
}
```

对于`TASK_EXECUTE_RESULT`事件，处理的子类是`TaskResultEventHandler`:

```java
@Component
public class TaskResultEventHandler implements TaskEventHandler {
	...
    @Override
    public void handleTaskEvent(TaskEvent taskEvent) throws TaskEventHandleError, TaskEventHandleException {
        int taskInstanceId = taskEvent.getTaskInstanceId();
        int processInstanceId = taskEvent.getProcessInstanceId();

        WorkflowExecuteRunnable workflowExecuteRunnable = this.processInstanceExecCacheManager.getByProcessInstanceId(
                processInstanceId);
        ...
        Optional<TaskInstance> taskInstanceOptional = workflowExecuteRunnable.getTaskInstance(taskInstanceId);
        ...
        TaskInstance taskInstance = taskInstanceOptional.get();
        if (taskInstance.getState().isFinished()) {
            sendAckToWorker(taskEvent);
            throw new TaskEventHandleError(
                    "Handle task result event error, the task instance is already finished, will discord this event");
        }
        ...
        TaskStateEvent stateEvent = TaskStateEvent.builder()
                .processInstanceId(taskEvent.getProcessInstanceId())
                .taskInstanceId(taskEvent.getTaskInstanceId())
                .status(taskEvent.getState())
                .type(StateEventType.TASK_STATE_CHANGE)
                .build();
        workflowExecuteThreadPool.submitStateEvent(stateEvent);

    }
```

除了修改任务状态，还构造了一个`TaskStateEvent`，发送到工作流实例的线程池，这是工作流的状态之一。

可以看到这里比较有意思，任务状态处理后，发送了触发工作流状态变化的事件。

`EventExecuteService`收到该事件后，交给`TaskStateEventHandler`处理，该方法会调用[DolphinScheduler-4：工作流的启动](https://izualzhy.cn/ds-how-process-start)里的`submitPostNode`继续提交下游任务。

## 4. 总结

Dolphin里有两处状态机的具体实现，一处是在任务状态，一处是在工作流状态。实现方式上都是类似的，既考虑了顺序性(准确性)又兼顾了性能，其中性能上主要是通过线程池同时确保单个工作流实例只有单个线程在处理。

状态机的作用之一，就是确保了 DAG 的算子能够依次顺利执行。

## 5. 参考资料
1. [Finite-State Machines: Theory and Implementation](https://code.tutsplus.com/finite-state-machines-theory-and-implementation--gamedev-11867t)   
