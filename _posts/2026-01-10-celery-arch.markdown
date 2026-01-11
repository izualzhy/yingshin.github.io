---
title: "任务队列 Celery 架构"
date: 2026-01-10 16:56:34
tags: microservice
---

## 1. 定位：Celery 是什么？

Celery 是一个分布式任务队列系统，用于在多个工作进程和机器之间**异步执行任务**。

异步任务的需求很多，特别是耗时且需要平滑处理的场景。比如文件格式转换、数据统计、调用第三方耗时的 api 等，就需要任务入队，然后逐批出队处理。

有些情况下，异步任务还往往伴随着延迟或者周期处理的需求，例如统计网站使用量、文件个数大小等，Celery 也还支持了**周期及延迟任务**。

## 2. 思考：如果自己实现

实现任务队列，基础是链条上参与的三个角色：**生产者、队列、消费者**  

首先是生产者：需要支持用户自定义任务，能够方便的提交到任务队列，队列里存储的应当是任务的元信息。这块需要提供代码框架或者接口实现。  

其次是队列：任务不丢的基础是持久化。我的第一个想法是采用消息队列存储，Celery 则支持了多种存储格式。  

最后是消费者：从队列中取出任务，实际执行用户任务代码。这里简单易用的实现方式是线程池，如果用户代码不可控、或者环境冲突，任务就需要放到容器执行。

基于上述基础功能，我们可能还要考虑很多**实际生产问题**：
1. 是否需要背压：类似流式管道，队列是否是有界的，当生产速度已经远远大于消费速度，是否应当让生产者感知？  
2. 不丢：比如消费者取走任务后，队列服务、消费者服务重启，此时是否会导致丢任务？
3. 不重：任务不能重复分发，如果是分布式系统下的典型 Unknown 场景，那此时是保证了 Exactly-once 还是 At-least-once ？大部分场景下我们追求后者，转而要求用户任务应当是幂等的。 
4. 失败重试：任务队列是否支持重试任务，还是让任务自身重试？如果任务自身，在任务队列系统看来跟普通执行并没有区别，可能会一直占着并发  
5. 延迟处理：任务队列能否指定该异步任务半小时后执行？  
6. 运行结果：如果是 fire and forget 的任务，不需要结果，可是如果是需要任务运行后的结果呢？类似于 akka 的 ? !  
7. 监控：当前有多少任务执行？多少排队？  

## 3. 架构：Celery 怎么实现的

```mermaid
graph TB
    subgraph "客户端层"
        ClientApp["客户端应用"]
        TaskDef["任务定义<br/>@app.task装饰器"]
    end
    
    subgraph "Celery应用"
        CeleryApp["Celery<br/>celery/app/base.py"]
        TaskRegistry["任务注册表<br/>celery/app/registry.py"]
        ConfigSystem["配置系统<br/>celery/app/defaults.py"]
        AMQP["AMQP层<br/>celery/app/amqp.py"]
        
        CeleryApp --> TaskRegistry
        CeleryApp --> ConfigSystem
        CeleryApp --> AMQP
    end
    
    subgraph "消息基础设施"
        Broker["消息代理<br/>RabbitMQ/Redis/SQS"]
        Queues["任务队列"]
        Router["消息路由器<br/>celery.app.routes"]
        
        Broker --> Queues
        AMQP --> Router
        Router --> Broker
    end
    
    subgraph "Worker系统"
        WorkerApp["Worker<br/>celery/apps/worker.py"]
        Consumer["消费者<br/>celery/worker/consumer.py"]
        Pool["执行池<br/>prefork/eventlet/gevent"]
        
        WorkerApp --> Consumer
        Consumer --> Pool
    end
    
    subgraph "结果存储"
        Backend["结果后端<br/>celery/backends/base.py"]
        RedisBackend["Redis后端"]
        RPCBackend["RPC后端"]
        DatabaseBackend["数据库后端"]
        
        Backend --> RedisBackend
        Backend --> RPCBackend
        Backend --> DatabaseBackend
    end
    
    subgraph "调度器系统"
        BeatApp["Beat<br/>celery/apps/beat.py"]
        Scheduler["调度器<br/>celery/beat.py"]
        
        BeatApp --> Scheduler
    end
    
    ClientApp --> TaskDef
    TaskDef --> CeleryApp
    
    Queues --> Consumer
    Pool --> Backend
    
    Scheduler --> AMQP
```
*注：图片来源 deepwiki<sup>1</sup>*

| 组件 | 用途 |
|------|------|
| Celery应用 | 嵌入在用户代码，支持方便的提交任务到队列  |
| 消息基础设施 | 外部服务，负责 队列管理、消息传递 |
| Worker | 进程管理、任务执行协调、并发任务执行（多进程/线程/异步  |
| 结果后端 | `celery/backends/base.py` | 结果持久化、状态跟踪 |
| Beat调度器 | `celery/apps/beat.py` | 周期性任务调度 |

用 Celery Github<sup>2</sup>上的极简例子说明上述组件在系统中的位置。

```
from celery import Celery

app = Celery('hello', broker='redis://localhost/0', backend='redis://localhost/0',)

@app.task
def hello():
    return 'hello world'
```

这里定义了
1. `app`定义了使用 Redis 作为任务的**消息基础设施**和**结果后端**  
2. 任务代码`hello`：跟普通定义没有差别，仅增加`@app.task`装饰

在实际使用时，执行`hello.delay`将任务提交到队列，即**Celery应用**。

然后通过`celery -A celery_app worker`的形式，启动**Worker**，`hello`方法的实际执行，也是在 Worker 里。

进一步的示例，我们放到下一篇笔记介绍。

## 4. 进一步思考：简洁的架构

在 Celery 里，任务的生命周期：

```mermaid
sequenceDiagram
    participant Client
    participant App as Celery App
    participant AMQP as AMQP Layer
    participant Broker
    participant Consumer
    participant Pool
    participant Backend
    
    Note over Client,Backend: 任务提交
    Client->>App: task.apply_async(args, kwargs)
    App->>App: 生成task_id (UUID)
    App->>AMQP: create_task_message()
    AMQP->>AMQP: 序列化args/kwargs/options
    AMQP->>Broker: 发布到队列
    
    Note over Client,Backend: 任务执行
    Broker->>Consumer: 投递消息
    Consumer->>Consumer: 反序列化消息
    Consumer->>Consumer: 确认消息
    Consumer->>Pool: 提交到工作池
    Pool->>Pool: 执行task.run()
    Pool->>Backend: store_result(task_id, result)
    
    Note over Client,Backend: 结果检索
    Client->>Backend: result.get(task_id)
    Backend->>Client: 返回结果或抛出异常
```
*注：图片来源 deepwiki<sup>1</sup>*

从任务的生命周期看，各个阶段参与的角色、职责非常清晰。

开始使用 Celery，用来对比的是大数据的任务调度系统，比如 Airflow DolphinScheduler 等，而 Celery 做到了**真正的去中心化**。我觉得核心原因在于周期任务，如果考虑任务调度，就需要引入 master 角色来实现准时且唯一的调度能力，而 Celery 的定位是任务队列，天然就避免了这一层。

当 Celery 通过 Celery Beat 支持周期任务后，这个问题就再次暴露出来，结果就是 Celery Beat 的单点风险。

同时通过任务生命周期可以看到，任务提交是在`Client`，而执行是在`Pool`，也就意味着：
1. `Client`至少需要看到 task 的声明
2. `Pool`需要看到 task 的定义

这个区别的影响在下一篇的例子里也会进一步说明。

这种使用方式让我想到 RPC 里 Client Server 的关系，比如 protobuffer 以及 proto 定义，两者都会用到，因此 RPC 的消息可以仅包含各类方法、实体名字即可，通过 PB 的兼容性确保了处理不会因为升级出错。

对应的经验，就是要**注意升级任务定义的兼容性**。比如`Client`里提交的 task 定义是`def task(file_path: str, type: int)`，而`Pool`执行的 task 在升级时变成了`def task(file_content: str, type: str)`，就会有问题，需要避免。

## 5. 参考资料

1. [deepwiki](https://deepwiki.com/celery/celery)
2. [celery](https://github.com/celery/celery)
