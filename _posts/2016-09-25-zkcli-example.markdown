---
title: 使用zkcli实现一个主-从模式的架构
date: 2016-9-25 21:59:19
excerpt: "使用zkcli实现一个主-从模式的架构"
tags: zookeeper
---

本文主要介绍通过zkCli工具实现主-从实例的一些功能，当然实际情况中我们永远不会这么去搭建系统。只是为了说明通过ZooKeeper，如何方便的实现多进程间的任务协作。

本文实例来源于[ZooKeeper：分布式过程协同技术详解](http://www.duokan.com/book/106575)。

<!--more-->

对一个主-从模式，主要参与者有：主节点、从节点、客户端。

分别负责了：  
1. 主节点：只有一个，分配客户端的任务到从节点，监视从节点的增删，以及任务完成的情况。同时系统在主节点挂掉时，需要有重新选举主节点的能力。  
2. 从节点：监视主节点分配给自己的任务并完成  
3. 客户端：创建新任务并等待系统的相应  

接下来介绍下如何通过多个zkCli-shell来实现上述功能。

### 1. 主节点

只有一个进程会成为主节点（多个主节点分环类似），因此需要通过加锁的方式确定主节点，未成为主节点的进程成为备用节点。当主节点崩溃时，接替主节点的角色。

这里我们启动两个zkCli-shell:zkCli1 + zkCli2模拟竞选主节点的两个进程，两个zkCli都同时尝试创建一个`/master`的ephemeral znode，只有一个创建成功，同时成为主节点。

```
#zkCli1创建成功，成为主节点，data表明所在进程
[zk: 127.0.0.1:2181(CONNECTED) 0] create -e /master "zkCli-1"
Created /master
[zk: 127.0.0.1:2181(CONNECTED) 1] ls /
[zookeeper, master]
```

```
#zkcli2创建失败，成为备用主节点
[zk: 127.0.0.1:2181(CONNECTED) 0] create -e /master "zkCli-2"
Node already exists: /master
#监控/master节点
[zk: 127.0.0.1:2181(CONNECTED) 1] stat /master true
cZxid = 0x1c5
ctime = Sun Sep 25 23:09:31 CST 2016
mZxid = 0x1c5
mtime = Sun Sep 25 23:09:31 CST 2016
pZxid = 0x1c5
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x1575ceca89a0013
dataLength = 7
numChildren = 0
```

zkCli1创建成功，成为主节点，zkCli2监控/master节点进入standby状态。

我们模拟主节点退出的情况，退出zkCli1，观察zkCli2能否监视到这种情况。

```
# 监视到/master的NodeDeleted
[zk: 127.0.0.1:2181(CONNECTED) 2] 
WATCHER::

WatchedEvent state:SyncConnected type:NodeDeleted path:/master

#创建/master成功，data表明所在进程
[zk: 127.0.0.1:2181(CONNECTED) 2] create -e /master "zkCli-2"
Created /master
```

zkCli1在重新启动后可以继续watch /master，就可以实现备主节点的作用。

选举完成主节点后，我们开始实现主从模式的一些基本功能：  
客户端分配任务后，主节点如何监测到并且分配任务到从节点执行任务，并在执行完成后如何通知客户端。

我们使用三个zkCli-shell分别代表主节点、从节点、客户端。

首先创建3个一层znode：`/workers /tasks /assign`，先简要介绍下各个znode的作用：  
1. `/workers`：从节点注册自身到该znode；主节点watch。  
2. `/tasks`：客户端在该znode下创建任务并watch该任务；主节点watch，并分配新的任务到从节点，从节点完成后建立任务完成标志以便客户端及时知道。  
3. `/assign`：从节点watch自己的任务分配子节点，主节点接收客户端任务后分配给从节点。  

接下来我们具体介绍下znode的操作，实现任务协作的整个过程。

### 2. 主节点关注从节点、任务

启动主节点zkCli

```
#关注从节点
[zk: 127.0.0.1:2181(CONNECTED) 0] ls /workers true
[]
#关注任务节点
[zk: 127.0.0.1:2181(CONNECTED) 1] ls /tasks true
[]
```

### 3. 从节点

从节点需要告诉主节点自己可以执行任务：通过在/workers子节点下创建临时的znode来通知主节点，并在子节点中使用主机名来标识自己。

在从节点zkCli里执行：

```
# 从节点创建worker1，value为自身ip:port
[zk: 127.0.0.1:2181(CONNECTED) 0] create -e /workers/worker1 "127.0.0.2:2222"
Created /workers/worker1
```

此时主节点zkCli会收到子节点建立的时间通知：

```
# 主节点接收到/workers下子节点变化的通知
WATCHER::
WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/workers
```

下一步，从节点zkcli创建`/assign/worker1`并监视这个节点的变化，等待新任务的到来

```
# 创建/assign/worker1接收任务分配
[zk: 127.0.0.1:2181(CONNECTED) 1] create /assign/worker1 ""
Created /assign/worker1
# 监视节点变化
[zk: 127.0.0.1:2181(CONNECTED) 2] ls /assign/worker1 true
[]
```

### 4. 客户端

客户端向系统添加任务，在客户端zkCli里执行

```
# 创建一个任务
[zk: 127.0.0.1:2181(CONNECTED) 0] create -s /tasks/task- "cmd"
Created /tasks/task-0000000000
```

可以看到我们添加了一个新的任务，多个任务类似队列的形式，因此我们创建的是sequential节点。同时客户端关注该任务是否已经完成：

```
# 关注任务是否完成
[zk: 127.0.0.1:2181(CONNECTED) 1] ls /tasks/task-0000000000 true
[]
```

创建任务时主节点会接收到这个事件，接下来我们开始看下整个任务创建后是如何协作完成的。

### 5. 任务协作

主节点收到任务创建的事件

```
WATCHER::
WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/tasks
```

于是检查新的任务、可用的子节点，然后分配（注意这里我们考虑简单情况，对任务是否已经分配、子节点是否空闲的问题都先忽略）

```
# 查看任务
[zk: 127.0.0.1:2181(CONNECTED) 2] ls /tasks
[task-0000000000]
# 查看可用子节点
[zk: 127.0.0.1:2181(CONNECTED) 3] ls /workers
[worker1]
# 分配任务到子节点
[zk: 127.0.0.1:2181(CONNECTED) 4] create /assign/worker1/task-0000000000 ""
Created /assign/worker1/task-0000000000
```


此刻子节点worker1可以接收到任务创建的消息：

```
WATCHER::
WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/assign/worker1
# 检查是否有新任务，并确认任务是否分配给自己
[zk: 127.0.0.1:2181(CONNECTED) 3] ls /assign/worker1
[task-0000000000]
# 执行任务...完成后在/tasks下添加一个状态znode
[zk: 127.0.0.1:2181(CONNECTED) 4] create /tasks/task-0000000000/statue "done"
Created /tasks/task-0000000000/statue
```

之后客户端接收到通知，并检查执行结果

```
WATCHER::
WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/tasks/task-0000000000
# 查看任务znode
[zk: 127.0.0.1:2181(CONNECTED) 2] get /tasks/task-0000000000
cmd
cZxid = 0x1d5
ctime = Mon Sep 26 22:24:18 CST 2016
mZxid = 0x1d5
mtime = Mon Sep 26 22:24:18 CST 2016
pZxid = 0x1d9
cversion = 1
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 3
numChildren = 1
# 查看任务执行结果
[zk: 127.0.0.1:2181(CONNECTED) 3] get /tasks/task-0000000000/statue
done
cZxid = 0x1d9
ctime = Mon Sep 26 22:55:18 CST 2016
mZxid = 0x1d9
mtime = Mon Sep 26 22:55:18 CST 2016
pZxid = 0x1d9
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 4
numChildren = 0
```

至此，一个分布式的任务成功执行完成了。在实际情况里，我们不会通过zkcli来完成，不过这种监视点通知回调的思路是比较通用的，包括临时节点、有序节点的使用。

### 5. 参考资料

1. [ZooKeeper：分布式过程协同技术详解](http://www.duokan.com/book/106575)
