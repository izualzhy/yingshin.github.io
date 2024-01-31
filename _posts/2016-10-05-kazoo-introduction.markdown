---
title: zookeeper-kazoo介绍
date: 2016-10-5 11:34:36
excerpt: "zookeeper-kazoo介绍"
tags: zookeeper
---

在[zookeeper的c客户端](http://izualzhy.cn/zookeeper-c-api-introduction)里，更多的是提供了食材，至于最后如何完成食谱(recipe)，则由使用方自己根据场景实现。因此经常给人一种接口很多，貌似实现了分布式的协调服务，但是却又需要自己实现一遍的感觉。

相比c++，java拥有更高级别的封装库：Curator。本文想要介绍的是python下非常好用的库：**Kazoo**。

Kazoo在使用上更加方便，同时对比Kazoo和c接口，也方便我们认识到zookeeper应用更高阶需求是什么，如果要实现一个更好用的c客户端，应该实现什么功能。

<!--more-->

废话不多说，看下Kazoo都有哪些feature(注意最后一条o(╯□╰)o)：

+ A wide range of recipe implementations, like Lock, Election or Queue
+ Data and Children Watchers
+ Simplified Zookeeper connection state tracking
+ Unified asynchronous API for use with greenlets or threads
+ Support for gevent 0.13 and gevent 1.0
+ Support for eventlet
+ Support for Zookeeper 3.3 and 3.4 servers
+ Integrated testing helpers for Zookeeper clusters
+ Pure-Python based implementation of the wire protocol, avoiding all the memory leaks, lacking features, and debugging madness of the C library

### 1. 安装

```
pip install kazoo
```

[源码地址](https://github.com/python-zk/kazoo)和[文档地址](http://kazoo.readthedocs.io/en/latest/index.html)。

### 2. 连接建立和断开

```python
from kazoo.client import KazooClient

zk = KazooClient(hosts='127.0.0.1:2181')
zk.start()
```

`zk.start()`同步的形式连接服务集群。

也可以使用异步的方式

```python
start_event = zk.asysc_start()
start_event.wait()
```

连接的状态可以查看`KazooClient.state`，Kazoo定义了三种状态来描述服务集群：
SUSPENDED CONNECTED LOST，通过`kazoo.client.KazooState`可以查看。跟c lib里`zookeeper_init`传入一个global的回调函数相同，我们也可以监视服务集群状态

```python
from kazoo.client import KazooState

def my_listener(state):
    if state == KazooState.LOST:
        # Register somewhere that the session was lost
    elif state == KazooState.SUSPENDED:
        # Handle being disconnected from Zookeeper
    else:
        # Handle being connected/reconnected to Zookeeper

zk.add_listener(my_listener)
```

listener可以有多个，使用`add_listener`添加即可，同时`remove_listener`支持移除listener。

当服务集群状态发生变化时，listener会逐个被调用。详细了解三种状态的转化，可以参考[understanding-kazoo-states](http://kazoo.readthedocs.io/en/latest/basic_usage.html#understanding-kazoo-states)

注意如果有报错："No handlers could be found for logger "kazoo.client"

在初始化之前配置log即可：

```python
import logging
logging.basicConfig()
```

### 3. CRUD

关于CRUD的操作，可以直接参考[kazoo相关文档](http://kazoo.readthedocs.io/en/latest/basic_usage.html)

#### 3.1 Creating

`create`用于创建节点，除了ACL, 临时节点，有序节点，还支持递归创建（`makepath=True`）

有的需求并不关注key对应的data，`ensure_path`在节点存在时返回True，不存在则递归创建该path。

比如

```python
zk.create('/ps/spider/dlb-receiver/0001')
```

#### 3.2 Reading

`exists`检查znode是否存在。

`get`获取节点数据`data`，以及属性`stat`。

`get_children`返回给定节点的子节点list。

#### 3.3 Updating

`set`更新znode节点data

#### 3.4 delete

`delete`删除znode，`recursive=True`支持递归删除。


### 4. 监视点

Kazoo提供了两种方式的监视。

以下几种接口支持设置监视点：

```python
get(path, watch=None)
get_children(path, watch=None, include_data=False)
exists(path, watch=None)
```

例如：

```python
def watch_func(event):
    ...

data, stat = zk.get("/my/favorite", watch=watch_func)
children = zk.get_children("/my/favorite", watch=watch_func)
stat = zk.exists("/my/favorite", watch=watch_func)
```

当"/my/favorite"发生关心的变化时，例如`get`，`exists`关心"/my/favorite"创建删除和data改变，`get_children`关心"/my/favorite"和子节点的创建删除。

以上set watch参数跟c客户端相同，都是单次触发。其实并不难理解，因为何时通知是在服务端确定的。但是借助python[装饰器](http://izualzhy.cn/python-decorator-notes)，Kazoo在客户端实现了触发后自动继续注册监视点的方式。
以上set watch参数跟c客户端相同，都是单次触发。其实并不难理解，因为何时通知是在服务端确定的。但是借助python[装饰器](http://izualzhy.cn/python-decorator-notes)，Kazoo在客户端实现了触发后自动继续注册监视点的方式。

```python
class kazoo.recipe.watchers.DataWatch(client, path, func=None, *args, **kwargs)
class kazoo.recipe.watchers.ChildrenWatch(client, path, func=None, allow_session_lost=True, send_event=False)
```

举个例子：

```python
@zk.ChildrenWatch('/my/favorite')
def watch_children(children):
    print 'watch_children', children

@zk.DataWatch('/my/favorite')
def watch_data(data, stat):
    print 'watch_data v1', data, stat

@zk.DataWatch('/my/favorite')
def watch_data(data, stat, event):
    print 'watch_data v2', data, stat, event
```

事件触发后，对应的函数例如`watch_children watch_data`会调用，并且自动继续注册该监视点。原理就是使用`KazooRetry + KazooClient.get`获取数据，同时watch指定为自定义函数，该自定义函数包装了我们真实的watch函数，并在每次自定义函数被回调时继续通过`KazooClient.get`注册。具体一些可以参考[源码](http://kazoo.readthedocs.io/en/latest/_modules/kazoo/recipe/watchers.html#DataWatch)，其中有很多decorator,partial,exception的处理方法，读起来也是非常有意思。

### 5. 事务

之前在[c客户端的介绍里]()提到过java的`multitop`的方式，多个修改在同一个事务里提交，保证了原子性，kazoo同样提供了`transaction`实现该功能。

```python
transaction = zk.transaction()
transaction.check('/node/a', version=3)
transaction.create('/node/b', b"a value")
results = transaction.commit()
```

### 6. 锁(lock)

应用中我们经常有多个进程抢锁的需求，使用原始的c lib我们经常采用竞争建立同一临时节点抢锁的形式。Kazoo提供了一组`lock`接口实现该功能。

```python
zk = KazooClient()
zk.start()
lock = zk.Lock("/lockpath", "my-identifier")
with lock:  # blocks waiting for lock acquisition
    # do something with the lock
```

`with lock`如果抢锁失败，会hold住。

identifier用于指定抢锁者的身份，`contenders()`获取当前所有的抢锁者。

注意本质上kazoo是通过在指定的path下建立临时子节点的形式，例如上面的代码建立了`/lockpath/afaef06196474594a59ce5dc082c3c03__lock__0000000004`，对应的data为'my-identifier'。

### 7. 选举(election)

有了锁的功能，很容易想到zookeeper另一个很重要的应用：群首选举(leader elections)，使用`kazoo.recipe.election`可以大大简化应用方的代码复杂度。

摘抄个例子：

```python
zk = KazooClient(hosts='127.0.0.1:2181')
zk.start()
# blocks until the election is won, then calls
# my_leader_function()
def leader_function():
    print 'enter leader_function'
    time.sleep(10)
    print 'after leader_function'

election = zk.Election("/electionpath", "identifier:%d" % os.getpid())
election.run(leader_function)
print election.contenders()
```

竞争群首失败执行`run`会hold住，成功则执行指定的leader_function。

`identifier`是该选举者的身份表示，通过`contenders()`接口可以获取到当前所有的群首竞争者。


### 8. 参考

1. [kazoo](http://kazoo.readthedocs.io/en/latest/index.html)
