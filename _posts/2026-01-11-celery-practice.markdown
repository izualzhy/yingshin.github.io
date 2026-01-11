---
title: "任务队列 Celery 实践"
date: 2026-01-11 11:09:51
tags: microservice
---

[上一篇](https://izualzhy.cn/celery-arch)介绍了 Celery 架构，这篇我们实战看看。

## 1. 例子

我从 xxx 几个方面，完整的例子放在了 github 上

### 1.1. Hello World

入门例子使用非常简洁，分为两步：
1. 启动Worker: 从 Redis 读取任务，执行`add`方法，将结果写回 Redis  
2. `run_simple_task -> add.delay`: 写入任务，通过`app`分发到 default 队列(存储到 Redis)，并等待读取结果  

#### 1.1.1. 消费者-Worker

先从启动 Worker 开始，定义`app/celery_app.py`：

```python
#!/usr/bin/env python
# coding=utf-8

from celery import Celery
from kombu import Queue, Exchange

app = Celery('demo_celery',
             broker='redis://localhost/0',
             backend='redis://localhost/0',
             include=['app.tasks'])
```

指定`app.celery_app`，启动 Worker ：

```bash
╰─$ export PYTHONPATH="../..:$PYTHONPATH"
../../.venv/bin/celery -A app.celery_app worker -l info -P threads -Q default,periodic_tasks -c 1


celery@yingzdeMacBook-Pro.local v5.5.3 (immunity)

macOS-10.16-x86_64-i386-64bit 2026-01-11 19:08:19

[config]
.> app:         demo_celery:0x10741d9a0
.> transport:   redis://localhost:6379/0
.> results:     redis://localhost/0
.> concurrency: 1 (thread)
.> task events: OFF (enable -E to monitor tasks in this worker)

[queues]
.> default          exchange=default(direct) key=default
.> periodic_tasks   exchange=periodic_tasks(direct) key=periodic_tasks

[tasks]
  . app.tasks.ack_task
  . app.tasks.add
  . app.tasks.bind_task
  . app.tasks.failing_task
  . app.tasks.log_message
  . app.tasks.no_ack_task
  . app.tasks.periodic_task
  . app.tasks.sqrt_task

[2026-01-11 19:08:19,574: INFO/MainProcess] Connected to redis://localhost:6379/0
[2026-01-11 19:08:19,577: INFO/MainProcess] mingle: searching for neighbors
[2026-01-11 19:08:20,582: INFO/MainProcess] mingle: all alone
[2026-01-11 19:08:20,597: INFO/MainProcess] celery@yingzdeMacBook-Pro.local ready.
```

#### 1.1.2. 生产者

首先需要定义任务本身，写法跟普通函数一样，唯一的区别在于装饰器里指定了 Celery 对象名，即上一节定义的`app = Celery(...)`：

```python
from py3_journey.demo_celery.app.celery_app import app

@app.task
def add(x, y):
    time.sleep(50)
    task_logger.info(f"Adding {x} and {y}")
    return x + y
```

然后调用`add.delay`方法

```python
def run_simple_task():
    logger.info("Dispatching simple 'add' task...")
    # using delay to trigger a task
    result = add.delay(4, 4)
    logger.info(f"Task dispatched. Task ID: {result.id}. Waiting for result...")
    logger.info(f"Result: {result.get(timeout=10)}")
```

执行`run_simple_task`就可以看到 Worker 的输出了：
```
[2026-01-11 21:17:55,407: INFO/MainProcess] Task app.tasks.add[4ae42872-1285-4599-b23a-f4f29f6dd7f9] received
2026-01-11 21:18:45,411 - celery.tasks - [ThreadPoolExecutor-0_0] - INFO - Adding 4 and 4
[2026-01-11 21:18:45,411: INFO/MainProcess] Adding 4 and 4
[2026-01-11 21:18:45,474: INFO/MainProcess] Task app.tasks.add[4ae42872-1285-4599-b23a-f4f29f6dd7f9] succeeded in 50.06529796200084s: 8
```

Worker 执行完成后，也就看到了 Result 的日志：
```
2026-01-11 21:17:55,275 - Dispatching simple 'add' task...
2026-01-11 21:17:55,407 - Task dispatched. Task ID: 4ae42872-1285-4599-b23a-f4f29f6dd7f9. Waiting for result...
2026-01-11 21:18:45,474 - Result: 8
```

从之前的分析知道`add`是在 Worker 执行的，所以**如果修改了`add`这类执行代码，需要重启/热加载 Worker**。

提交任务的方式有多种：
1. `app.send_task`: 任务名作为字符串形式，好处是解耦，不需要在一个代码库 ，缺点则是 task 改名、找到谁在调用等运维问题   
2. `task.apply_async()`: 标准方式，有 task 对象；   
3. `task.delay()`: apply_async 的简化版，简洁、不能指定复杂参数    
4. `task.apply`: 在当前线程执行，不经过 broker worker，主要用于调试场景    

### 1.2. 失败重试任务

普通场景下 Celery 的任务方法，实现与普通函数一样，但是随着深入，比如重试的写法就不同。

```python
@app.task(bind=True, max_retries=3, default_retry_delay=60)
def failing_task(self):
    try:
        task_logger.info(f"This task will fail and retry, self.request : {self.request}")
        raise ValueError("Intentional error")
    except ValueError as exc:
        task_logger.warning(f"Task failed. Retrying in {self.default_retry_delay} seconds...")
        raise self.retry(exc=exc)
```

上面是一个重试方法的例子，区别在于指定了`bind=True`，以及采用`self.retry`重试。

如果是自定义函数，可能会采用`@retry`或者 sleep 以达到重试的效果，但是这样**最大的问题是占着 Worker 并发**。

`self.retry`并不会立即重试，而是增加重试时间后，重新推回了队列。

执行上述代码，观察 Worker 的日志：

```
[2026-01-11 21:43:01,135: INFO/MainProcess] Task app.tasks.failing_task[4f524ae5-9df0-47bd-b3a7-2ad5e683ab35] received
2026-01-11 21:43:01,135 - celery.tasks - [ThreadPoolExecutor-0_0] - INFO - This task will fail and retry, self.request : ... 'id': '4f524ae5-9df0-47bd-b3a7-2ad5e683ab35', ...
[2026-01-11 21:43:01,135: INFO/MainProcess] This task will fail and retry, self.request : ... 'id': '4f524ae5-9df0-47bd-b3a7-2ad5e683ab35', 'delivery_tag': '31633d28-5ee1-4051-a25c-be4b878059cd' ...
2026-01-11 21:43:01,136 - celery.tasks - [ThreadPoolExecutor-0_0] - WARNING - Task failed. Retrying in 60 seconds...
[2026-01-11 21:43:01,136: WARNING/MainProcess] Task failed. Retrying in 60 seconds...
[2026-01-11 21:43:01,174: INFO/MainProcess] Task app.tasks.failing_task[4f524ae5-9df0-47bd-b3a7-2ad5e683ab35] received
[2026-01-11 21:43:01,182: INFO/MainProcess] Task app.tasks.failing_task[4f524ae5-9df0-47bd-b3a7-2ad5e683ab35] retry: Retry in 60s: ValueError('Intentional error')

2026-01-11 21:44:01,142 - celery.tasks - [ThreadPoolExecutor-0_0] - INFO - This task will fail and retry, self.request : ... 'id': '4f524ae5-9df0-47bd-b3a7-2ad5e683ab35', 'delivery_tag': '31633d28-5ee1-4051-a25c-be4b878059cd' ...
```

可以看到任务 `id=4f524ae5-9df0-47bd-b3a7-2ad5e683ab35` 不变，重试是通过 Celery 的机制完成，**因此重试期间不会占用 Worker 并发**。

由于我们当前使用的是 Redis Broker，从 Redis 也可以大概猜测其实现方式：

```bash
127.0.0.1:6379> zrange unacked_index 0 -1 withscores
1) "31633d28-5ee1-4051-a25c-be4b878059cd"
2) "1.7681390411489222e+9"
```

delivery_tag 的 score 正好对应了任务重试时，下次执行的时间。

### 1.3. bind 任务

上一节里可以看到`bind=True`的参数，该参数的效果使得原本函数本身支持了上下文，因此也就可以调用`self.retry`.

bind 之后的函数可以获取到更多上下文参数，可以用于自定义的场景：

```python
@app.task(bind=True)
def bind_task(self, i):
    task_logger.info(f'i : {i} , self.request : {self.request}')
```

常用的有：

| 属性                           | 含义              |
| ---------------------------- | --------------- |
| `self.request.id`            | 当前 task id      |
| `self.request.retries`       | 已重试次数           |
| `self.request.hostname`      | 哪个 worker 在跑    |
| `self.request.argsrepr`          | 原始参数            |
| `self.request.kwargsrepr`        | 原始 kwargs       |
| `self.request.delivery_info` | queue / routing |

### 1.4. 如何保证任务不丢

保证任务不丢的实现方式，在于**任务信息应当何时从 Broker 删除？**，是 Worker 取走任务还是 Worker 执行完成任务。

答案显然是后者，我们可以用如下例子来验证：

```python
@app.task
def no_ack_task():
    task_logger.info("This task will not be acknowledged.")
    time.sleep(30)
    task_logger.info("No acknowledgment task completed successfully.")
    return "No acknowledgment"

@app.task(acks_late=True)
def ack_task():
    task_logger.info("This task demonstrates the acknowledgment mechanism.")
    time.sleep(30)
    task_logger.info("Ack task completed successfully.")
    return "Acknowledged"
```

启动 Worker(至少两个并发)，提交上述任务，通过日志验证任务已经在 Worker 执行。

此时重启 Worker，只有`ack_task`会重试。

当然这里只能确保 AtLeastOnce 的语义，比如 Worker 闪断导致判定失败，就会存在重复执行，此时则依赖`ack_task`在实现上是幂等的。

### 1.5. 周期任务

Celery 是通过 Beat 支持定时任务的，启动方式：

```bash
#!/bin/bash
# run_beat.sh
echo "Starting Celery beat scheduler..."

export PYTHONPATH="../..:$PYTHONPATH"
../../.venv/bin/celery -A app.celery_app beat -l info
```

Beat 的原理，其实就是充当了生产者的角色，按照周期时间将任务提交的队列，Beat 本身不执行任务。

基于此设计，**Beat 只能是单点执行**。

## 2. 可观测

**一个系统从功能可用，到生产环境可用，中间还要经过 N 多验证，比如性能测试、容错测试、第三方依赖故障测试等、以及可观测性的评估**。

很多时候我们可能因为时间问题直接跳过，但是作业终究还是需要补的。

Celery 提供了一些常用命令，用于了解任务队列状态，例如：

```bash
celery -A bisheng.worker.main inspect active       # 查看当前正在执行的任务
celery -A bisheng.worker.main inspect reserved     # 查看已接收但尚未执行（预取）的任务
celery -A bisheng.worker.main inspect scheduled    # 查看 ETA／countdown 延时任务
celery -A bisheng.worker.main inspect active_queues # 查看每个 worker 所在队列及消费情况
```

如果未调度（即还在 Broker 里），则需要查看对应的 Broker 存储。

举个例子，比如如果队列积压，任务可能存在于两处：
1. Celery 已分配给 Worker  
2. Celery 未分配给 Worker  

前者查看：

```bash
╰─$ celery -A app.celery_app inspect reserved
->  celery@yingzdeMacBook-Pro.local: OK
    * {'id': '8a649961-0e21-4278-8788-f8fe369f46e4', 'name': 'app.tasks.add', 'args': [1, 1], 'kwargs': {}, 'type': 'app.tasks.add', 'hostname': 'celery@yingzdeMacBook-Pro.local', 'time_start': None, 'acknowledged': False, 'delivery_info': {'exchange': '', 'routing_key': 'default', 'priority': 0, 'redelivered': False}, 'worker_pid': None}
    * {'id': '49a33ae1-44cd-4d83-a99a-5dbe85f2fd96', 'name': 'app.tasks.add', 'args': [2, 2], 'kwargs': {}, 'type': 'app.tasks.add', 'hostname': 'celery@yingzdeMacBook-Pro.local', 'time_start': None, 'acknowledged': False, 'delivery_info': {'exchange': '', 'routing_key': 'default', 'priority': 0, 'redelivered': False}, 'worker_pid': None}
    * {'id': 'ba4cfb3c-c46d-475d-9700-b9ee00d022ff', 'name': 'app.tasks.add', 'args': [4, 4], 'kwargs': {}, 'type': 'app.tasks.add', 'hostname': 'celery@yingzdeMacBook-Pro.local', 'time_start': None, 'acknowledged': False, 'delivery_info': {'exchange': '', 'routing_key': 'default', 'priority': 0, 'redelivered': False}, 'worker_pid': None}
    * {'id': '5a43dfee-db9f-4144-9ede-061e11321620', 'name': 'app.tasks.add', 'args': [3, 3], 'kwargs': {}, 'type': 'app.tasks.add', 'hostname': 'celery@yingzdeMacBook-Pro.local', 'time_start': None, 'acknowledged': False, 'delivery_info': {'exchange': '', 'routing_key': 'default', 'priority': 0, 'redelivered': False}, 'worker_pid': None}
```

后者则需要直接查看 Redis：

```bash
127.0.0.1:6379> lrange default 0 -1
1) "{\"body\": \"W1s5LCA5XSwge30sIHsiY2FsbGJhY2tzIjogbnVsbCwgImVycmJhY2tzIjogbnVsbCwgImNoYWluIjogbnVsbCwgImNob3JkIjogbnVsbH1d\", \"content-encoding\": \"utf-8\", \"content-type\": \"application/json\", \"headers\": {\"lang\": \"py\", \"task\": \"app.tasks.add\", \"id\": \"919ecb65-33e3-4f91-a6ea-2f060f79135a\", \"shadow\": null, \"eta\": null, \"expires\": null, \"group\": null, \"group_index\": null, \"retries\": 0, \"timelimit\": [null, null], \"root_id\": \"919ecb65-33e3-4f91-a6ea-2f060f79135a\", \"parent_id\": null, \"argsrepr\": \"(9, 9)\", \"kwargsrepr\": \"{}\", \"origin\": \"gen13764@yingzdeMacBook-Pro.local\", \"ignore_result\": false, \"replaced_task_nesting\": 0, \"stamped_headers\": null, \"stamps\": {}}, \"properties\": {\"correlation_id\": \"919ecb65-33e3-4f91-a6ea-2f060f79135a\", \"reply_to\": \"d1f96e44-82f3-3ea5-b912-5909a6f90f42\", \"delivery_mode\": 2, \"delivery_info\": {\"exchange\": \"\", \"routing_key\": \"default\"}, \"priority\": 0, \"body_encoding\": \"base64\", \"delivery_tag\": \"b0b55f87-981f-443b-bcf8-679ca2e8b159\"}}"
2) "{\"body\": \"W1s4LCA4XSwge30sIHsiY2FsbGJhY2tzIjogbnVsbCwgImVycmJhY2tzIjogbnVsbCwgImNoYWluIjogbnVsbCwgImNob3JkIjogbnVsbH1d\", \"content-encoding\": \"utf-8\", \"content-type\": \"application/json\", \"headers\": {\"lang\": \"py\", \"task\": \"app.tasks.add\", \"id\": \"643637da-27a5-44d5-9930-8bd8bed72cee\", \"shadow\": null, \"eta\": null, \"expires\": null, \"group\": null, \"group_index\": null, \"retries\": 0, \"timelimit\": [null, null], \"root_id\": \"643637da-27a5-44d5-9930-8bd8bed72cee\", \"parent_id\": null, \"argsrepr\": \"(8, 8)\", \"kwargsrepr\": \"{}\", \"origin\": \"gen13764@yingzdeMacBook-Pro.local\", \"ignore_result\": false, \"replaced_task_nesting\": 0, \"stamped_headers\": null, \"stamps\": {}}, \"properties\": {\"correlation_id\": \"643637da-27a5-44d5-9930-8bd8bed72cee\", \"reply_to\": \"d1f96e44-82f3-3ea5-b912-5909a6f90f42\", \"delivery_mode\": 2, \"delivery_info\": {\"exchange\": \"\", \"routing_key\": \"default\"}, \"priority\": 0, \"body_encoding\": \"base64\", \"delivery_tag\": \"ccbac1d6-a32b-41b4-997c-8603616d6e3c\"}}"
3) "{\"body\": \"W1s3LCA3XSwge30sIHsiY2FsbGJhY2tzIjogbnVsbCwgImVycmJhY2tzIjogbnVsbCwgImNoYWluIjogbnVsbCwgImNob3JkIjogbnVsbH1d\", \"content-encoding\": \"utf-8\", \"content-type\": \"application/json\", \"headers\": {\"lang\": \"py\", \"task\": \"app.tasks.add\", \"id\": \"756de473-96bf-4643-95a0-702b96b6afae\", \"shadow\": null, \"eta\": null, \"expires\": null, \"group\": null, \"group_index\": null, \"retries\": 0, \"timelimit\": [null, null], \"root_id\": \"756de473-96bf-4643-95a0-702b96b6afae\", \"parent_id\": null, \"argsrepr\": \"(7, 7)\", \"kwargsrepr\": \"{}\", \"origin\": \"gen13764@yingzdeMacBook-Pro.local\", \"ignore_result\": false, \"replaced_task_nesting\": 0, \"stamped_headers\": null, \"stamps\": {}}, \"properties\": {\"correlation_id\": \"756de473-96bf-4643-95a0-702b96b6afae\", \"reply_to\": \"d1f96e44-82f3-3ea5-b912-5909a6f90f42\", \"delivery_mode\": 2, \"delivery_info\": {\"exchange\": \"\", \"routing_key\": \"default\"}, \"priority\": 0, \"body_encoding\": \"base64\", \"delivery_tag\": \"dea6405f-3a88-4f5c-9e55-102a319f29aa\"}}"
4) "{\"body\": \"W1s2LCA2XSwge30sIHsiY2FsbGJhY2tzIjogbnVsbCwgImVycmJhY2tzIjogbnVsbCwgImNoYWluIjogbnVsbCwgImNob3JkIjogbnVsbH1d\", \"content-encoding\": \"utf-8\", \"content-type\": \"application/json\", \"headers\": {\"lang\": \"py\", \"task\": \"app.tasks.add\", \"id\": \"e3f3cc57-62da-4a97-8a05-a099a0a0e49b\", \"shadow\": null, \"eta\": null, \"expires\": null, \"group\": null, \"group_index\": null, \"retries\": 0, \"timelimit\": [null, null], \"root_id\": \"e3f3cc57-62da-4a97-8a05-a099a0a0e49b\", \"parent_id\": null, \"argsrepr\": \"(6, 6)\", \"kwargsrepr\": \"{}\", \"origin\": \"gen13764@yingzdeMacBook-Pro.local\", \"ignore_result\": false, \"replaced_task_nesting\": 0, \"stamped_headers\": null, \"stamps\": {}}, \"properties\": {\"correlation_id\": \"e3f3cc57-62da-4a97-8a05-a099a0a0e49b\", \"reply_to\": \"d1f96e44-82f3-3ea5-b912-5909a6f90f42\", \"delivery_mode\": 2, \"delivery_info\": {\"exchange\": \"\", \"routing_key\": \"default\"}, \"priority\": 0, \"body_encoding\": \"base64\", \"delivery_tag\": \"2decdf7b-153b-409e-beb3-b4a3bc396472\"}}"
5) "{\"body\": \"W1s1LCA1XSwge30sIHsiY2FsbGJhY2tzIjogbnVsbCwgImVycmJhY2tzIjogbnVsbCwgImNoYWluIjogbnVsbCwgImNob3JkIjogbnVsbH1d\", \"content-encoding\": \"utf-8\", \"content-type\": \"application/json\", \"headers\": {\"lang\": \"py\", \"task\": \"app.tasks.add\", \"id\": \"7e96b5a6-2972-4931-829c-4a5d29be5ec9\", \"shadow\": null, \"eta\": null, \"expires\": null, \"group\": null, \"group_index\": null, \"retries\": 0, \"timelimit\": [null, null], \"root_id\": \"7e96b5a6-2972-4931-829c-4a5d29be5ec9\", \"parent_id\": null, \"argsrepr\": \"(5, 5)\", \"kwargsrepr\": \"{}\", \"origin\": \"gen13764@yingzdeMacBook-Pro.local\", \"ignore_result\": false, \"replaced_task_nesting\": 0, \"stamped_headers\": null, \"stamps\": {}}, \"properties\": {\"correlation_id\": \"7e96b5a6-2972-4931-829c-4a5d29be5ec9\", \"reply_to\": \"d1f96e44-82f3-3ea5-b912-5909a6f90f42\", \"delivery_mode\": 2, \"delivery_info\": {\"exchange\": \"\", \"routing_key\": \"default\"}, \"priority\": 0, \"body_encoding\": \"base64\", \"delivery_tag\": \"dad2f2be-4f8a-4c71-a647-5c650a1d6b05\"}}"
```

这种方式操作不便，同时可以看到跟 Broker 类型强绑定。

Celery 通过提供 **flower 解决了状态可视化的问题**。

```
╰─$ celery -A app.celery_app flower

[I 260111 22:54:13 command:168] Visit me at http://0.0.0.0:5555
[I 260111 22:54:13 command:176] Broker: redis://localhost:6379/0
[I 260111 22:54:13 command:177] Registered tasks:
    ['app.tasks.ack_task',
     'app.tasks.add',
     'app.tasks.bind_task',
     'app.tasks.failing_task',
     'app.tasks.log_message',
     'app.tasks.no_ack_task',
     'app.tasks.periodic_task',
     'app.tasks.sqrt_task',
     'celery.accumulate',
     'celery.backend_cleanup',
     'celery.chain',
     'celery.chord',
     'celery.chord_unlock',
     'celery.chunks',
     'celery.group',
     'celery.map',
     'celery.starmap']
[I 260111 22:54:13 mixins:228] Connected to redis://localhost:6379/0
```

启动后，通过 flower 页面，可以方便的看到 Worker Tasks Broker 的情况。  

![](/assets/images/celery/flower_web.png)

比如当我们一次性推送较多任务到 Broker，可以看到任务的情况：

![](/assets/images/celery/flower_web_2.png)

对于尚未分发出来的任务，则可以通过 Broker 页面看到积压的任务数：

![](/assets/images/celery/flower_web_broker.png)

状态可视化解决了，那异常时如何报警？比如当队列数超过 1000 则认为系统出现问题。

这块不得不说 Prometheus 协议的强大之处，flower 同时启动了 <http://0.0.0.0:5555/metrics> 地址提供了符合 Prometheus 抓取格式的数据，因此就可以天然对接 Prometheus 了:

![](/assets/images/celery/celery_flower_metrics.png)

当然这里如果想要放到生产环境，需要确保只采集单个 flower 实例的指标数据。

## 3. 其他

我们使用 Celery 最直接的是 queue，但是实际上观察前面打印的 delivery_info ，可以看到包含了 routing_key exchange 等信息，

```
task
  └─ routing_key
        ↓
     exchange
        ↓（匹配规则）
      queue
        ↓
      worker
```

routing_key 的设计好处，在于提供了一个配置 routing_key -> queue 的映射关系。

比如我们希望处理不同 level 的日志，写入不同的 queue

如果是自己维护的话，可能是这么写：
```
process_log.apply_async(queue=f'{level}_logs')
```

但是采用`routing_key`的这种配置化写法，则灵活性更高，能应对更复杂的情况。

```
   process_log.apply_async(exchange='logs', routing_key=level, ...)

   task_queues=(
       Queue('error_logs',   logs_exchange, routing_key='error'),
       Queue('warning_logs', logs_exchange, routing_key='warning'),
  
       # 将 'info' 和 'debug' 这两个不同的路由键，都指向同一个队列
       Queue('transient_logs', logs_exchange, routing_key='info'),
       Queue('transient_logs', logs_exchange, routing_key='debug'),
   )
```

此外我在研究 Celery 时，发现不同的 backend，由于特性不同，支持的能力也有差别，例如 reject_on_worker_lost 等参数；以及如何指定任务的优先级、如何结合其他系统实现动态队列、动态 Worker 等，这些是否还是 Celery 系统所擅长解决的？就需要进一步研究了。
