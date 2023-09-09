---
title: "DolphinScheduler笔记之5: 普通任务CommonTaskProcessor"
date: 2023-09-09 00:52:37
tags: [DolphinScheduler-3.1.3]
---

接[上一篇笔记](https://izualzhy.cn/dolphinscheduler-process-start)，这里总计下普通任务在 master 的处理过程。

DolphinScheduler 里的任务类型，按照逻辑可以分为两种：  
1. 普通任务：具体执行的任务，例如 Shell、SQL、Flink 等，相当于编程语言里的函数、计算    
2. 条件分支：用于判断下一个任务是否执行，例如 Dependent、Conditions、Switch 等，相当于编程语言里的 while/for/if     

对于普通任务，master 打包发送到 worker 执行；对于逻辑分支，master 交给自身执行。

显然这两种类型的任务处理过程是不同的：前者分发到 worker，需要考虑负载均衡、字段协议、网络延迟等；后者在 master 执行，需要考虑线程隔离、CPU占用。

同时两者又都有相同点：保存任务状态、失败重试、触发下游节点等。

DolphinScheduler 通过封装`BaseTaskProcessor`实现了两者在处理流程上的统一。

## 1. BaseTaskProcessor 类的思路

任务执行过程分为多个阶段：提交任务、分发任务、运行任务、停止任务等，这些行为定义到了枚举类`enum TaskAction`.

`BaseTaskProcessor.action`方法接收该参数:

```java
public abstract class BaseTaskProcessor implements ITaskProcessor {
	@Override
	public boolean action(TaskAction taskAction) {
		...
		switch (taskAction) {
			case STOP:
				result = stop();
			case RUN:
				result = run();
			case DISPATCH:
				result = dispatch();
			...
		}
	}
}
```

action 实际最后调用了 `runTask` `dispatchTask`:

```java

protected boolean run() {
        return runTask();
    }

    protected boolean dispatch() {
        return dispatchTask();
    }
    protected abstract boolean runTask();
    protected abstract boolean dispatchTask();
```

因此，各个子类按需重载对应的抽象方法就可以了，典型的子类例如`CommonTaskProcessor`、`DependentTaskProcessor`。

具体介绍`CommonTaskProcessor`的实现前，先看看外部是如何调用`action`的。

## 2. BaseTaskProcessor 的调用

taskProcessor 的初始化和主要调用是在上篇笔记提到的`WorkflowExecuteRunnable.submitTaskExec`方法：

```java
public class WorkflowExecuteRunnable implements Callable<WorkflowSubmitStatue> {

    private Optional<TaskInstance> submitTaskExec(TaskInstance taskInstance) {
            ITaskProcessor taskProcessor = TaskProcessorFactory.getTaskProcessor(taskInstance.getTaskType());
            taskProcessor.init(taskInstance, processInstance);
            ...	
            boolean submit = taskProcessor.action(TaskAction.SUBMIT);
            ...
            boolean dispatchSuccess = taskProcessor.action(TaskAction.DISPATCH);
            ...
            taskProcessor.action(TaskAction.RUN);

```

调用顺序：`init` -> `action(SUBMIT)` -> `action(DISPATCH)` -> `action(RUN)`   

因此重点看看`CommonTaskProcessor`里这几个方法的实现。

## 3. CommonTaskProcessor 的实现

### 3.1. init

`CommonTaskProcessor`继承自`BaseTaskProcessor`，`init`方法在基类实现，主要是初始化各个成员变量：

例如：   
+ `processService`提供了工作流实例、任务实例的更新接口，同时支持修改数据库。    
+ `processInstanceDao`定义在这里比较突兀，只有`SubTaskProcessor`使用，正常数据库统一通过`processService`更新比较合适。     


```java
    public void init(@NonNull TaskInstance taskInstance, @NonNull ProcessInstance processInstance) {
        processService = SpringApplicationContext.getBean(ProcessService.class);
        processInstanceDao = SpringApplicationContext.getBean(ProcessInstanceDao.class);
        masterConfig = SpringApplicationContext.getBean(MasterConfig.class);
        taskPluginManager = SpringApplicationContext.getBean(TaskPluginManager.class);
        curingParamsService = SpringApplicationContext.getBean(CuringParamsService.class);
        this.taskInstance = taskInstance;
        this.processInstance = processInstance;
        this.maxRetryTimes = masterConfig.getTaskCommitRetryTimes();
        this.commitInterval = masterConfig.getTaskCommitInterval().toMillis();
    }
```

### 3.2. submitTask

这里的 submit 是指到 DB，调用的核心实现在`ProcessServiceImpl.submitTaskInstanceToDB`，更新了 taskInstance：
包括`executorId`、`state`、`submitTime`、`firstSubmitTime`属性

**更新 DB 在调度系统里是非常重要的一步**，主要两个作用：
1. taskInstance 的全生命周期管理，例如状态包含 SUBMITTED_SUCCESS、RUNNING_EXECUTION、SUCCESS、DISPATCH 等  
2. 持久化以确保异常时任务的 recovery  

```java
public class ProcessServiceImpl implements ProcessService {

    @Override
    public TaskInstance submitTaskInstanceToDB(TaskInstance taskInstance, ProcessInstance processInstance) {
        WorkflowExecutionStatus processInstanceState = processInstance.getState();
        if (processInstanceState.isFinished() || processInstanceState == WorkflowExecutionStatus.READY_STOP) {
            logger.warn("processInstance: {} state was: {}, skip submit this task, taskCode: {}",
                    processInstance.getId(),
                    processInstanceState,
                    taskInstance.getTaskCode());
            return null;
        }
        if (processInstanceState == WorkflowExecutionStatus.READY_PAUSE) {
            taskInstance.setState(TaskExecutionStatus.PAUSE);
        }
        taskInstance.setExecutorId(processInstance.getExecutorId());
        taskInstance.setState(getSubmitTaskState(taskInstance, processInstance));
        if (taskInstance.getSubmitTime() == null) {
            taskInstance.setSubmitTime(new Date());
        }
        if (taskInstance.getFirstSubmitTime() == null) {
            taskInstance.setFirstSubmitTime(taskInstance.getSubmitTime());
        }
        boolean saveResult = saveTaskInstance(taskInstance);
        if (!saveResult) {
            return null;
        }
        return taskInstance;
    }
```

### 3.3. dispatchTask

这一步将待分发任务放到了`TaskPriorityQueueImpl`队列，也就是[DolphinScheduler笔记之4：工作流的启动](https://izualzhy.cn/dolphinscheduler-process-start)图里的`TaskPriorityQueue`

该队列通过 Bean 装配：

```java
    public void initQueue() {
        this.taskUpdateQueue = SpringApplicationContext.getBean(TaskPriorityQueueImpl.class);
    }
```

队列元素为`TaskPriority`对象，顾名思义，对象包含两部分：

+ Priority 相关：

例如`processInstancePriority`、`processInstanceId`、`taskInstancePriority`、`taskGroupPriority`、`taskInstanceId`、`groupName`、`checkpoint`等，其中`checkpoint`是指对象生成的时间，生成越早优先级越高  

初始化传入的各参数来源：

```java
            TaskPriority taskPriority = new TaskPriority(processInstance.getProcessInstancePriority().getCode(),
                    processInstance.getId(), taskInstance.getProcessInstancePriority().getCode(),
                    taskInstance.getId(), taskInstance.getTaskGroupPriority(),
                    Constants.DEFAULT_WORKER_GROUP);
```

+ Task 相关：

例如`taskName`、`scheduleTime`、`taskType`、`taskParams`、`tenantCode`这些通用信息；以及具体任务类型相关信息，比如对于 SQL 任务，任务SQL、数据源的访问串都会在这一步准备好。   

注意这点设计非常重要，**确保了 Worker 模块不需要也不会访问 dolphin 的数据库。**   

Task 相关信息封装到了类`TaskExecutionContext`，

### 3.4. TaskPriorityQueueConsumer

`TaskPriorityQueueConsumer`是一个线程类，`run`的核心实现在`batchDispatch`，从上一节写入的队列里取出`taskPriority`，放到线程池`consumerThreadPoolExecutor`执行`dispatchTask`方法：

```java
public class TaskPriorityQueueConsumer extends BaseDaemonThread {
    public List<TaskPriority> batchDispatch(int fetchTaskNum) throws TaskPriorityQueueException, InterruptedException {
        List<TaskPriority> failedDispatchTasks = Collections.synchronizedList(new ArrayList<>());
        CountDownLatch latch = new CountDownLatch(fetchTaskNum);

        for (int i = 0; i < fetchTaskNum; i++) {
            TaskPriority taskPriority = taskPriorityQueue.poll(Constants.SLEEP_TIME_MILLIS, TimeUnit.MILLISECONDS);
            if (Objects.isNull(taskPriority)) {
                latch.countDown();
                continue;
            }

            consumerThreadPoolExecutor.submit(() -> {
                try {
                    boolean dispatchResult = this.dispatchTask(taskPriority);
                    if (!dispatchResult) {
                        failedDispatchTasks.add(taskPriority);
                    }
                } finally {
                    // make sure the latch countDown
                    latch.countDown();
                }
            });
        }

        latch.await();

        return failedDispatchTasks;
    }

```

交给线程池执行是非常必要的，主要原因是分发任务到 worker 是 IO 操作，非常耗时。     

`dispatchTask`的核心代码如下：
+ `toCommand`方法将`context`封装到`TaskDispatchCommand`，再转为`Command`.该类是 master worker 之间 RPC 的基础数据结构。   
+ `dispatcher.dispatch`方法将`executionContext`发送到 worker  
+ `addDispatchEvent`记录分发事件  


```java
    protected boolean dispatchTask(TaskPriority taskPriority) {
        ...
            TaskExecutionContext context = taskPriority.getTaskExecutionContext();
            ExecutionContext executionContext =
                    new ExecutionContext(toCommand(context), ExecutorType.WORKER, context.getWorkerGroup(),
                            taskInstance);

            ...

            result = dispatcher.dispatch(executionContext);

            if (result) {
                logger.info("Master success dispatch task to worker, taskInstanceId: {}, worker: {}",
                        taskPriority.getTaskId(),
                        executionContext.getHost());
                addDispatchEvent(context, executionContext);
            } else {
                logger.info("Master failed to dispatch task to worker, taskInstanceId: {}, worker: {}",
                        taskPriority.getTaskId(),
                        executionContext.getHost());
            }

```

### 3.5. ExecutorDispatcher.dispatch

这里主要有两步：
1. 负载均衡就不多说了，主要是选取合适的 host 发送，具体可以查看`HostManager`的子类实现。不过从我的实际经验看，LowWeight负载均衡效果一般，主要还是 Worker 任务一般反射弧都比较长。    
2. `executorManager.execute`发送任务到 worker     

```java
public class ExecutorDispatcher implements InitializingBean {
    public Boolean dispatch(final ExecutionContext context) throws ExecuteException {
        ...
        // host select
        Host host = hostManager.select(context);
        ...
        context.setHost(host);
        executorManager.beforeExecute(context);
        try {
            // task execute
            return executorManager.execute(context);
        } finally {
            executorManager.afterExecute(context);
        }
    }

```

### 3.6. NettyExecutorManager.execute

dolphin 使用 Netty 收发数据，可以看到发送的数据只有`context.command`，如果发送失败，则尝试发送到其他 host.   
`doExecute`方法比较简单，就是通过 channel 发送数据，失败时重试。  

```java
public class NettyExecutorManager extends AbstractExecutorManager<Boolean> {
    public Boolean execute(ExecutionContext context) throws ExecuteException {
        ...
        // build command accord executeContext
        Command command = context.getCommand();
        // execute task host
        Host host = context.getHost();
        boolean success = false;
        while (!success) {
            try {
                doExecute(host, command);
                success = true;
                context.setHost(host);
                // We set the host to taskInstance to avoid when the worker down, this taskInstance may not be
                // failovered, due to the taskInstance's host
                // is not belongs to the down worker ISSUE-10842.
                context.getTaskInstance().setHost(host.getAddress());
            } catch (ExecuteException ex) {
                ...
                        host = Host.of(remained.iterator().next());
            }
        }

        return success;
    }
```

## 4. 总结

master 构建完 DAG 生成 TaskInstance 后，`CommonTaskProcessor`完成了后续分发任务的部分。重点需要考虑状态持久化、负载均衡、网络协议、线程隔离等。

`BaseTaskProcessor`作为基类，封装了任务在 master 的执行过程。了解该过程有助于了解各个子类的实现，比如`DependentTaskProcessor`，接下来的笔记会继续总结该类的实现逻辑。
