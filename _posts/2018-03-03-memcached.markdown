---
title: "memcache十分钟入门"
date: 2018-03-03 12:49:21
excerpt: "memcache"
tags: distribute
---

## 1. 简介

memcache是一种典型的C/S架构服务，对外提供基于key-value映射的快速数据存储和检索服务。常见应用场景例如用户session信息、状态数据、共享数据等。

单机缓存的一个典型问题就是数据共享问题，为了提高单机缓存的命中率，我们经常采用分环的做法，使得相同数据总是落到一个实例上。然而分环又带来了数据热点问题，在线架构分环难度更大。如果使用统一缓存，多机共享可以跳过这个问题。同时memcache采用全内存存储，提高了数据读写的速度。

注意memcache虽然称为分布式缓存，但memcache本身并不是分布式的，而是一个单机缓存系统。memcache集群之间不会通信（交换数据），分布式的实现依赖于客户端对集群的分环以及双写随机读这类架构设计。

本文介绍下memcache的常用操作。

<!--more-->

## 2. 数据操作

memcache启动比较简单，`memcached -p 11211 -m 64m -d`就在后台以11211端口启动了。

```
$ memcached -p 11211 -m 64m -d
$ netstat -anp | grep 11211
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 0.0.0.0:11211               0.0.0.0:*                   LISTEN      10612/memcached
udp        0      0 0.0.0.0:11211               0.0.0.0:*                               10612/memcached
```

memcache支持文本协议，因此我们使用telnet来演示下，`telnet 0 11211`就可以连接上memcached。其中0表示本机地址，11211是memcached使用的端口。

数据操作的介绍，自然要从CRUD讲起。

### 2.1. set/get

数据的新增和更新使用`set`

```
set hello 123 5 5
world
STORED
```

上面我们存储了键值对`hello -> world`，后端返回存储成功`STORED`.

`set`的语法为

```
set key flags exptime bytes [noreply]
value
```

其中参数含义为：

|args  |notes  |
|--|--|
|key  | k-v结构里的key  |
|flags  | 上下文信息，整型  |
|exptime  | 缓存时间（秒  |
|bytes  | value大小  |
|value  | k-v结构里的value（**始终位于第二行**）  |

上面的例子里，`123`是我们的上下文信息（必须为整型），第一个`5`是cache有效时间，第二个`5`表示value的大小。

`set`之后就可以使用`get`获取数据了

```
get hello
VALUE hello 123 5
world
END
```

`get`多个key使用空格分隔，例如`get key1 key2 ...`

注意：
1. get不会导致expired time重新计时
2. 命令行计时是从第一行enter后开始的

### 2.2. add

`add` 的语义是 insert，跟 set 类似，不过如果 add 的 key 已经存在，则不会更新数据(过期的 key 会更新)，之前的值将仍然保持相同，此时返回 NOT_STORED。

因此实际架构应用里会设计用于抢锁。

```
set hello 123 30 5
world
STORED

add hello 123 30 5
gocpp
NOT_STORED

add name 123 30 4
Jeff
STORED
```

可以看到`set hello`之后，尝试`add hello`失败。

### 2.3. replace

`replace`的语义是 update，用于替换已存在的 key - value。跟 set 类似，跟 add 正好相反。如果 key 不存在，则替换失败，返回 NOT_STORED。

```
set hello 123 30 5
world
STORED

replace hello 321 10 5
hahah
STORED

get hello
VALUE hello 321 5
hahah
END

replace name 123 10 4
Jeff
NOT_STORED
```

可以看到`replace hello`返回 STORED，`replace name`失败，返回 NOT_STORED.

### 2.4. append/prepend

append命令用于向已存在 key 的 value 后面追加数据。

```
set hello 123 10 5
world
STORED

append hello 123 10 5
world
STORED

get hello
VALUE hello 123 10
worldworld
END
```

`append hello`后对原 value 起到一个追加的效果，value 修改为`worldworld`

如果要 append 的 key 不存在，返回 NOT_STORED.

`prepend` 跟 `append` 行为相反，用于向已存在的 key 的 value 前面追加数据。

### 2.5. gets/cas

`gets/cas`提供了类似事务的操作，`gets`返回一个类似于版本号的数值，称为token。`cas`更新数据时需要提供对应的token，否则更新失败。`cas`成功更新数据后，会更新该 token。

例如我们首先写入键值对：`hello -> world`。

`gets`的语法跟`get`一致，使用`gets`获取`hello`对应的 value：

```
gets hello
VALUE hello 123 5 26
world
END
```

返回值里最后一列数字代表了 hello 的 token：26

`cas`的语法跟`set`一致，只是第一行最后需要增加 token：

```
cas hello 456 0 5 26
hahah
STORED
```

如果`token`错误或者没有，返回`ERROR`:

```
cas hello 456 0 5
ERROR
```

如果中间有人更新了数据，返回`EXISTS`.例如我们在`cas`之前更新了数据:

```
gets hello
VALUE hello 123 5 28
hahah
END

set hello 123 0 5
hello
STORED

get hello
VALUE hello 123 5
hello
END

cas hello 123 0 5 28
hahah
EXISTS
```

可以看到`cas`再拿着`28`提交数据时，后端返回`EXISTS`.

`cas`的全称是 Compare And Swap，很多数据库都提供了这种操作，之前的[智能指针](https://izualzhy.cn/protobuf-shared_ptr#23-googleprotobufinternalweak_ptr)笔记里提到了内存操作的用法`NoBarrier_CompareAndSwap`。

### 2.6. delete

```
delete hello
DELETED

delete key_not_exist
NOT_FOUND
```

### 2.7. incr/decr

`incr/decr`用于计数的场景，例如我们统计domain下链接个数、接收的数据包个数等。

```
set cnt 123 0 2
99
STORED

incr cnt 1
100

decr cnt 50
50
```

## 3. 客户端

`c`客户端连接 memcached 的方案可以使用 [libMemcached](http://libmemcached.org/libMemcached.html)。

不过我们这里介绍下使用 [baidu-rpc](https://github.com/brpc/brpc/blob/master/docs/cn/memcache_client.md) 的方案。

```
#include "baidu/rpc/memcache.h"
#include "baidu/rpc/channel.h"

int main() {
    baidu::rpc::Channel channel;
    baidu::rpc::ChannelOptions options;
    options.protocol = baidu::rpc::PROTOCOL_MEMCACHE;

    if (channel.Init("0.0.0.0:11211", &options) != 0) {
        LOG(FATAL) << "Fail to init channel to memcached";
        return -1;
    }

    baidu::rpc::MemcacheRequest request;
    baidu::rpc::MemcacheResponse response;
    baidu::rpc::Controller cntl;
    if (!request.Set("hello", "world", 0x1234, 10, 0)) {
        LOG(FATAL) << "Fail to Set request";
        return -1;
    }

    channel.CallMethod(NULL, &cntl, &request, &response, NULL);
    if (cntl.Failed()) {
        LOG(FATAL) << "Fail to access memcached, " << cntl.ErrorText();
        return -1;
    }

    if (!response.PopSet(NULL)) {
        LOG(FATAL) << "Fail to Set memecached, " << response.LastError();
        return -1;
    }

    return 0;
}
```

## 4. twempoxy

前面提到了 memcache 要想实现分布式的功能，需要客户端自己实现 hash 分环，因此很多 memcache 的实际应用里会使用 proxy。

[twemproxy](https://github.com/twitter/twemproxy) 是 twitter 提供的访问 redis/memcache 的轻量级 proxy，相比客户端自己实现 hash 分环，主要有三个优点：
1. 与后端 memcache server保持一个长连接，避免 client 直连 cache server, 提高通信效率
2. 一致性 hash 路由规则：将请求均匀的分散到后端 server 上。同时后端 server 挂了时，避免命中率明显下降
3. 自动屏蔽 failure servers

用户可以像访问单实例一样，访问 memcache 集群。

安装方式也比较简单

```
$ cd twemproxy
$ autoreconf -fvi
$ ./configure --enable-debug=full
$ make
```

安装完成后，修改配置文件：`conf/nutcrack.yml`，例如：

```
delta:
  listen: 127.0.0.1:22124
  hash: fnv1a_64
  distribution: ketama
  timeout: 100
  auto_eject_hosts: true
  server_retry_timeout: 2000
  server_failure_limit: 1
  servers:
   - 127.0.0.1:11214:1
   - 127.0.0.1:11215:1
   - 127.0.0.1:11216:1
   - 127.0.0.1:11217:1
   - 127.0.0.1:11218:1
   - 127.0.0.1:11219:1
   - 127.0.0.1:11220:1
   - 127.0.0.1:11221:1
   - 127.0.0.1:11222:1
   - 127.0.0.1:11223:1
```

该配置监听22124端口，后端的memcached pool包含10个实例:`127.0.0.1:11214 ... 127.0.0.1:11223`。

启动这10个`memcached`实例（端口11214 .. 11223），然后启动`nutcracker`：

```
$ ./src/nutcracker
[2018-03-04 10:56:55.666] nc.c:189 nutcracker-0.4.0.2015.09.07 built for Linux 2.6.32_1-16-0-0_virtio x86_64 started on pid 16667
[2018-03-04 10:56:55.666] nc.c:194 run, rabbit run / dig that hole, forget the sun / and when at last the work is done / don't sit down / it's time to dig another one
[2018-03-04 10:56:55.688] nc_core.c:44 max fds 10240 max client conns 10197 max server conns 11
[2018-03-04 10:56:55.688] nc_stats.c:870 m 5 listening on '0.0.0.0:22222'
[2018-03-04 10:56:55.688] nc_proxy.c:218 p 8 listening on '127.0.0.1:22124' in memcache pool 0 'delta' with 10 servers
[2018-03-04 10:56:55.688] nc_core.c:159 whitelist:conf/authip interval:10
[2018-03-04 10:57:05.689] nc_ipwhitelist.c:90 Get mtime of whitelist file failed, possibly file does not exist
```

之后我们就可以像访问单实例一样访问该集群了：

```
$ telnet 0 22124
Trying 0.0.0.0...
Connected to 0.
Escape character is '^]'.
set hello 123 0 5
world
STORED

get hello
VALUE hello 123 5
world
END
```

数据实际上按照指定的hash算法存储在了某个实例上：

```
$ (echo 'get hello' && sleep 1) | telnet 0 11217
Trying 0.0.0.0...
Connected to 0.
Escape character is '^]'.
VALUE hello 123 5
world
END
```

## 5. reference

1. [memcache-tutorial](http://www.runoob.com/memcached/memcached-tutorial.html)
2. [twemproxy](https://github.com/twitter/twemproxy)
