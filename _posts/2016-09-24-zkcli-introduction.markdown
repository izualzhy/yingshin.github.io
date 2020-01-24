---
title: zookeeper之zkCli的使用
date: 2016-9-24 23:25:08
excerpt: "zookeeper之zkCli的使用"
tags: [zookeeper]
---

从本节开始，我们进入zk的实战部分。

zk客户端的封装除了提供各语言如JAVA/C/PYTHON的API，还提供了客户端的shell工具：发布在`bin`下的zkCli.sh。

使用zkCli.sh可以很方便对zookeeper进行调试和管理，包括创建、更新、删除znode，设置监视点等。同时交互式shell的方式不依赖库的编译，也非常适合初学者上手，本文主要介绍`zkCli.sh`下对znode的各种基本操作。


<!--more-->

### 1. zkcli启动

上一节中介绍了四字命令，其中`stat`可以显示当前的连接情况。  
在使用`zkCli.sh`连接前，我们先看下当前的状态：

```
[spider@cp01-saverng-2 bin]$ echo stat | nc 127.0.0.1 2181
Zookeeper version: 3.4.8--1, built on 02/06/2016 03:18 GMT
Clients:
 /127.0.0.1:48113[0](queued=0,recved=1,sent=0)

Latency min/avg/max: 0/0/22
Received: 6532
Sent: 6531
Connections: 1
Outstanding: 0
Zxid: 0x199
Mode: standalone
Node count: 25
```

接着使用zkCli连接到zkServer：`sh zkCli.sh -server host:port`

```
[spider@cp01-saverng-2 bin]$ sh zkCli.sh -server 127.0.0.1:2181
Connecting to 127.0.0.1:2181
Welcome to ZooKeeper!
JLine support is enabled

WATCHER::

WatchedEvent state:SyncConnected type:None path:null
[zk: 127.0.0.1:2181(CONNECTED) 0] 
```

可以看到我们收到了一个`SyncConnected`的Event，并且连接到了server(状态为CONNECTED)。

此时继续使用`stat`四字命令查看连接情况，会发现多了一个链接。

接下来介绍下znode的操作部分

### 2. zkCli基础

进入zkCli-shell后，`help`可以看到所有的命令及支持的参数，这里列举几个常用的命令。

#### 2.1 ls

查看节点下的子节点:`ls path [watch]`

```
# 查看根目录/下的子节点，zookeeper为自带节点
[zk: 127.0.0.1:2181(CONNECTED) 31] ls /
[zookeeper]
```

#### 2.2 create

创建节点，格式为`create [-s] [-e] path data acl`，如果value为空，填空字符串即可。

例如我们创建一个新的节点

```
[zk: 127.0.0.1:2181(CONNECTED) 6] create /testnode "testvalue"
Created /testnode
```

此时使用`ls`查看可以看到根节点下有两个子节点：`[zookeeper, testnode]`

上一节提到了节点有多种类型，`create`命令默认创建persistent节点，如果要创建sequential epheremal节点对应的命令为`-s -e`，例如我们创建/testnode的有序子节点

```
[zk: 127.0.0.1:2181(CONNECTED) 8] create -s /testnode/child-node- ""
Created /testnode/child-node-0000000000
```

创建后的节点名为`child-node-0000000000`

ephemeral在zkcli退出后一段时间会自动删除，读者可以自行验证下，并思考下这个自动删除的延迟时间的大小。

#### 2.3 get

获取节点值以及属性：`get path [watch]`

```
[zk: 127.0.0.1:2181(CONNECTED) 12] get /testnode
testvalue
cZxid = 0x1a3
ctime = Sun Sep 25 17:30:16 CST 2016
mZxid = 0x1a3
mtime = Sun Sep 25 17:30:16 CST 2016
pZxid = 0x1a5
cversion = 2
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 9
numChildren = 2
```

可以看到对应的data值为testvalue，get获取的属性称为Stat，各个字段的含义如下：

```
czxid. 节点创建时的zxid.
mzxid. 节点最新一次更新发生时的zxid.
ctime. 节点创建时的时间戳.
mtime. 节点最新一次更新发生时的时间戳.
dataVersion. 节点数据的更新次数.
cversion. 其子节点的更新次数.
aclVersion. 节点ACL(授权信息)的更新次数.
ephemeralOwner. 如果该节点为ephemeral节点, ephemeralOwner值表示与该节点绑定的session id. 如果该节点不是ephemeral节点, ephemeralOwner值为0.
dataLength. 节点数据的字节数.
numChildren. 子节点个数.
```

关于zxid的定义：ZooKeeper状态的每一次改变, 都对应着一个递增的Transaction id, 该id称为zxid. 由于zxid的递增性质, 如果zxid1小于zxid2, 那么zxid1肯定先于zxid2发生. 创建任意节点, 或者更新任意节点的数据, 或者删除任意节点, 都会导致Zookeeper状态发生改变, 从而导致zxid的值增加.

各个字段的含义都比较明确，ACL在以后会逐渐介绍到。

#### 2.4 set

对已存在的值进行更新：`set path data [version]`

```
[zk: 127.0.0.1:2181(CONNECTED) 15] set /testnode "testvalue_new"
cZxid = 0x1a3
ctime = Sun Sep 25 17:30:16 CST 2016
mZxid = 0x1a6
mtime = Sun Sep 25 21:22:21 CST 2016
pZxid = 0x1a5
cversion = 2
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 13
numChildren = 2
```

可以使用`get`查看data的值是否更新，同时看到stat.dataVersion由0变为1。  
`dataVersion`即为znode的版本号，`set delete`两个操作都可以传递版本号作为参数，只有当传入参数的存在与服务器上的版本号一致时调用才会成功，不一致则返回错误`version No is not valid`。当需要阻止并行操作带来的不一致性时，可以考虑使用版本号实现。

#### 2.5 stat

查看节点的属性stat：`stat path [watch]`

```
[zk: 127.0.0.1:2181(CONNECTED) 16] stat /testnode
cZxid = 0x1a3
ctime = Sun Sep 25 17:30:16 CST 2016
mZxid = 0x1a6
mtime = Sun Sep 25 21:22:21 CST 2016
pZxid = 0x1a5
cversion = 2
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 13
numChildren = 2
```

#### 2.6. delete

删除节点：`delete path [version]`

注意如果有子节点，需要使用`rmr`。

注意上面的命令都可以使用非交互的方式，`-server host:port cmd args`

例如

```
sh zkCli.sh -server xxx:2181 ls /
```

### 3. 监视

zkCli支持对znode添加监视点，通过help可以看到：

```
stat path [watch]
ls path [watch]
get path [watch]
```

最后一个参数传入true，表示除了原来的行为，还在该path建立一个监视点。

举个`ls`命令的例子：

```
#ls最后一个参数=true，表示关注该节点的删除事件、子节点事件
[zk: 127.0.0.1:2181(CONNECTED) 27] ls /testnode true
[child-node-0000000001, child-node-0000000000]
#创建一个新的seq节点
[zk: 127.0.0.1:2181(CONNECTED) 28] create -s /testnode/child-node- ""
Created /testnode/child-node-0000000002
WATCHER::

#收到通知，state = SyncConnected type = NodeChildrenChanged
WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/testnode
```

`stat get`可以类似的创建监视点,关于对应的`type state`我们在C-API一节里会更加详细的介绍。

### 4. 参考资料
1. [ZooKeeper：分布式过程协同技术详解](http://www.duokan.com/book/106575)
