---
title: zookeeper C API介绍
date: 2016-10-2 21:28:08
excerpt: "zookeeper C API介绍"
tags: zookeeper
---

整体上，zookeeper提供的C API可以分为三种：

1. 辅助接口：例如连接的初始化和销毁，状态的获取等  
2. 同步接口：同步操作znode，zoo_xxx/zoo_wxxx的形式，例如创建(zoo_create），查看节点data(zoo_get/zoo_wget)等  
3. 异步接口：异步操作znode，zoo_axxx的形式，例如创建(zoo_acreate)，查看节点data(zoo_aget)等  

<!--more-->

在接口设计上zookeeper非常灵活，例如查看znode是否存在，提供了接口`zoo_exists/zoo_wexists/zoo_aexists/zoo_awexists`，由应用方按照具体场景选择使用。

为了避免只是单纯的翻译介绍`zookeeper.h`，本文主要从整体上介绍这些接口的异同点，并提供了一些仅供学习环境的例子，以及总结的注意事项。zookeeper客户端代码遵守doxy规范，使用doxygen可以得到一份详细的接口文档，强烈建议读一遍文档。

我们从如何创建和销毁跟服务集群的连接开始，逐步介绍zookeeper客户端的接口使用。

### 1. 连接的创建和销毁

在使用ZooKeeper进行任何操作之前，需要一个`zhandle_t`句柄，用于管理客户端与服务器之间的连接，所有的zookeeper接口函数都需要传入该参数。通过`zookeeper.h`的两个接口进行创建和销毁:

```cpp
ZOOAPI zhandle_t *zookeeper_init(const char *host, watcher_fn fn,
  int recv_timeout, const clientid_t *clientid, void *context, int flags);

ZOOAPI int zookeeper_close(zhandle_t *zh);
```

`zookeeper_init`初始化一条与服务器之间的连接。

各个参数的含义：  
1. host: 服务集群主机地址的字符串，地址格式为host:port，每组地址以逗号分隔。  
2. fn：回调函数地址，如果设置了监视点，znode节点发生相关变化后，该回调函数会被调用。  
3. recv_timeout：会话过期时间，单位毫秒。  
4. clientid:之前已建立的一个会话的客户端ID，用于客户端重新连接。在建立会话时，通过调用`zoo_client_id`来获取客户端ID。指定参数0开始新的会话。  
5. context:传入一个上下文对象，在回调函数`fn`中使用，也可以通过`zoo。  
6. flags:暂时未使用，设置为0.  

回调函数声明如下：

```cpp
typedef void (*watcher_fn)(zhandle_t *zh, int type, 
        int state, const char *path,void *watcherCtx);
```

注意连接的创建是异步的，也就是当`zookeeper_init`返回时连接并不一定建立，只有在`fn`回调函数调用时返回`state = ZOO_CONNECTED_STATE`时才连接成功。

一个会话的生命周期是指会话从创建到结束的时期，中间可能的状态有：**CONNECTED CONNECTING CLOSED NOT_CONNECTED**。  

一个会话从NOT_CONNECTED开始，当ZooKeeper客户端初始化后转换到CONNECTING。成功建立连接后状态为CONNECTED，关闭连接后状态为CLOSED。中间可能断开连接状态变为CONNECTING。

状态以及可能状态的转换如下图：

![zk_status.png](/assets/images/zk_status.png)

`zookeeper_close`比较简单,用于释放已经申请的资源。

看一个简单的例子，在这个例子里我们连接zk服务集群，并在连接成功后断开。

```cpp
#include <assert.h>
#include <pthread.h>
#include <string>
#include <iostream>
#include "zookeeper.h"

const std::string server = "127.0.0.1:2181";

const std::string root_path = "/";
const int32_t session_timeout_ms = 15000;

volatile bool connected = false;

pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;

//连接后发送连接成功的signal
void signal_on_request_finished() {
    pthread_mutex_lock(&lock);
    connected = true;
    pthread_cond_signal(&cond);
    pthread_mutex_unlock(&lock);
}

//等待连接成功
void wait_until_connected() {
    pthread_mutex_lock(&lock);
    while (!connected) {
        pthread_cond_wait(&cond, &lock);
    }
    pthread_mutex_unlock(&lock);
}

//处理事件的监视点函数
void zk_event_callback(
        zhandle_t* zh,
        int type,
        int state,
        const char* path,
        void* watcherCtx) {
    std::cout << __PRETTY_FUNCTION__ << std::endl;
    std::cout << "zh:" << zh << std::endl;
    std::cout << "type:" << type << std::endl;
    std::cout << "state:" << state << std::endl;
    std::cout << "path:" << path << std::endl;
    std::string* context = static_cast<std::string*>(watcherCtx);
    std::cout << "watcherCtx:" << *context << std::endl;
    delete context;

    signal_on_request_finished();
}

zhandle_t *init_zhandle() {
    std::cout << __PRETTY_FUNCTION__ << std::endl;
    std::string* str = new std::string(__PRETTY_FUNCTION__);
    return zookeeper_init(
            (server + root_path).c_str(),
            zk_event_callback,
            15000,
            NULL,
            str,
            0);
}

void close_zhandle(zhandle_t*& handle) {
    std::cout << __PRETTY_FUNCTION__ << std::endl;
    int res = zookeeper_close(handle);
    std::cout << res << "\t" << zerror(res) << std::endl;
    handle = NULL;
}

int main() {
    zhandle_t *handle = init_zhandle();//连接zk服务端
    assert(handle != NULL);
    wait_until_connected();//等待连接成功
    close_zhandle(handle);//关闭连接，释放资源

    return 0;
}
```

程序调用`zookeeper_init`后等待`zk_event_callback`回调函数连接成功，然后调用`zookeeper_close`关闭连接。

分析下程序的输出：

```
zhandle_t* init_zhandle()
2016-10-02 23:03:40,844:5470(0x7f6957f39740):ZOO_INFO@log_env@674: Client environment:zookeeper.version=zookeeper C client 3.3.3.900
2016-10-02 23:03:40,844:5470(0x7f6957f39740):ZOO_INFO@log_env@678: Client environment:host.name=******
2016-10-02 23:03:40,844:5470(0x7f6957f39740):ZOO_INFO@log_env@685: Client environment:os.name=Linux
2016-10-02 23:03:40,844:5470(0x7f6957f39740):ZOO_INFO@log_env@686: Client environment:os.arch=2.6.32_1-17-0-0
2016-10-02 23:03:40,844:5470(0x7f6957f39740):ZOO_INFO@log_env@687: Client environment:os.version=#1 SMP Mon Aug 24 11:14:27 CST 2015
2016-10-02 23:03:40,844:5470(0x7f6957f39740):ZOO_INFO@log_env@695: Client environment:user.name=******
2016-10-02 23:03:40,844:5470(0x7f6957f39740):ZOO_INFO@log_env@703: Client environment:user.home=******
2016-10-02 23:03:40,844:5470(0x7f6957f39740):ZOO_INFO@log_env@715: Client environment:user.dir=******
2016-10-02 23:03:40,844:5470(0x7f6957f39740):ZOO_INFO@zookeeper_init@743: Initiating client connection, host=127.0.0.1:2181/ sessionTimeout=15000 watcher=0x402dfd sessionId=0 sessionPasswd=<null> context=0x617080 flags=0
2016-10-02 23:03:40,846:5470(0x7f695653a700):ZOO_INFO@check_events@1710: initiated connection to server [127.0.0.1:2181]
2016-10-02 23:03:40,851:5470(0x7f695653a700):ZOO_INFO@check_events@1757: session establishment complete on server [127.0.0.1:2181], sessionId=0x1575ceca89a0025, negotiated timeout=15000
void zk_event_callback(zhandle_t*, int, int, const char*, void*)
zh:0x619570
type:-1
state:3
path:
watcherCtx:zhandle_t* init_zhandle()
void close_zhandle(zhandle_t*&)
2016-10-02 23:03:40,851:5470(0x7f6957f39740):ZOO_INFO@zookeeper_close@2440: Closing zookeeper sessionId=0x1575ceca89a0025 to [127.0.0.1:2181]

0       ok
```


zookeeper客户端会输出部分日志，这里重点看下程序自身的输出：  
`zk_event_callback`里，`type = -1; state = 3`都是常量定义，分别为`ZOO_SESSION_EVENT; ZOO_CONNECTED_STATE`，此时表示连接确实建立。同时`zookeeper_close`是有返回值的，这里的值是0，我们使用`zerror`打印出了对应的字符串解释为"ok"。

因此，接下来我们先介绍下`type`,`state`,返回值等常见的常量， 以及监视点函数，也就是我们的全局回调函数。

### 2. 监视点函数、常量定义

监视点函数的定义如下：

```cpp
typedef void (*watcher_fn)(zhandle_t *zh, int type, 
        int state, const char *path,void *watcherCtx);
```

各参数说明：

1. 监视点函数引用的ZooKeeper句柄  
2. 事件类型，比如上面连接成功type=-1，对应ZOO_SESSION_EVENT  
3. 连接状态，比如上面连接成功state=3，对应ZOO_CONNECTED_STATE  
4. 被观察并触发时间的znode节点路径，如果时间为会话事件，路径为NULL  
5. 监视点上下文  

参数`type`相关的数值有：

```
ZOO_CREATED_EVENT:1
ZOO_DELETED_EVENT:2
ZOO_CHANGED_EVENT:3
ZOO_CHILD_EVENT:4
ZOO_SESSION_EVENT:-1
ZOO_NOTWATCHING_EVENT:-2
```

参数`state`相关的数值有：

```
ZOO_EXPIRED_SESSION_STATE:-112
ZOO_AUTH_FAILED_STATE:-113
ZOO_CONNECTING_STATE:1
ZOO_ASSOCIATING_STATE:2
ZOO_CONNECTED_STATE:3
```

返回值为`ZOO_ERRORS`连接的enum常量，常用的有`ZOK ZSYSTEMERROR ZRUNTIMEINCONSISTENCY`等，使用`ZOOAPI const char* zerror(int c)`可以转化成对应的字符串解释。

其他一些ACL常量、create flags等常量，我们在相关的API里根据具体使用环境介绍。

接下来我们介绍znode操作的同步接口，同步接口又可以分为两类，分别是zoo_xxx和zoo_wxxx的形式，例如:

```cpp
ZOOAPI int zoo_exists(zhandle_t *zh, const char *path, int watch, struct Stat *stat);
ZOOAPI int zoo_wexists(zhandle_t *zh, const char *path,
        watcher_fn watcher, void* watcherCtx, struct Stat *stat);
```

两者作用相同：都是通过传入`zh`来监控`path`这个znode，返回值表示是否存在或者其他错误。区别是在监视点的注册上，传入"watch flag"还是"watcher object"，这两个名词是zookeeper的术语，简言之就是传入一个int值，还是回调函数`watcher`+上下文`watcherCtx`来监视`path`，如果传入前者，则在`path`被创建时回调`zookeeper_init`里的回调函数，如果传入后者，则后者被回调。

也就是说`watch_fn`在两种情况下作为参数指定：`zookeeper_init`和`zoo_wxxx`类型的函数。

类似的接口还有`zoo_get/zoo_wget zoo_get_children/zoo_wget_children zoo_get_children2/zoo_wget_children2`

我们使用`zoo_exists/zoo_wexists`的例子来具体看下两种回调方式的区别。

### 3. 监视点回调的例子

```cpp
...

void zk_event_callback(
        zhandle_t* zh,
        int type,
        int state,
        const char* path,
        void* watcherCtx) {
    std::cout << __PRETTY_FUNCTION__ << std::endl;
    std::cout << "zh:" << zh << std::endl;
    std::cout << "type:" << type << std::endl;
    std::cout << "state:" << state << std::endl;
    std::cout << "path:" << path << std::endl;
    std::cout << "watcherCtx:" << watcherCtx << std::endl;
    std::string* context = static_cast<std::string*>(watcherCtx);
    std::cout << "watcherCtx:" << *context << std::endl;

    signal_on_request_finished();
}

void watch_event_callback(
        zhandle_t* zh,
        int type,
        int state,
        const char* path,
        void* watcherCtx) {
    std::cout << __PRETTY_FUNCTION__ << std::endl;
    std::cout << "zh:" << zh << std::endl;
    std::cout << "type:" << type << std::endl;
    std::cout << "state:" << state << std::endl;
    std::cout << "path:" << path << std::endl;
    std::cout << "watcherCtx:" << watcherCtx << std::endl;
    std::string* context = static_cast<std::string*>(watcherCtx);
    std::cout << "watcherCtx:" << *context << std::endl;

    signal_on_request_finished();
}

zhandle_t *init_zhandle() {
    std::cout << __PRETTY_FUNCTION__ << std::endl;
    std::string* str = new std::string(__PRETTY_FUNCTION__);//memory leak, use shared_ptr instead
    return zookeeper_init(
            (server + root_path).c_str(),
            zk_event_callback,
            15000,
            NULL,
            str,
            0);
}

void close_zhandle(zhandle_t*& handle) {
    std::cout << __PRETTY_FUNCTION__ << std::endl;
    int res = zookeeper_close(handle);
    std::cout << res << "\t" << zerror(res) << std::endl;
    handle = NULL;
}

void wexists(zhandle_t* zh, const char* path) {
    std::cout << __PRETTY_FUNCTION__ << std::endl;
    std::string* str = new std::string(__PRETTY_FUNCTION__);//memory leak, use shared_ptr instead
    int res = zoo_wexists(zh, path, watch_event_callback, str, NULL);
    std::cout << "res[" << res << "]:"<< zerror(res) << std::endl;
}

void exists(zhandle_t* zh, const char* path) {
    std::cout << __PRETTY_FUNCTION__ << std::endl;
    std::string* str = new std::string(__PRETTY_FUNCTION__);//memory leak, use shared_ptr instead
    zoo_set_context(zh, str);
    int res = zoo_exists(zh, path, 1, NULL);
    std::cout << "res[" << res << "]:"<< zerror(res) << std::endl;
}

void create(zhandle_t* zh, const char* path) {
    std::cout << __PRETTY_FUNCTION__ << std::endl;
    std::string data = "data_by_API_zoo_create";
    char buffer[64] = {0};
    int res = zoo_create(zh, path, data.c_str(), data.size(),
            // &ZOO_OPEN_ACL_UNSAFE, ZOO_SEQUENCE, buffer, sizeof(buffer));
            &ZOO_OPEN_ACL_UNSAFE, ZOO_EPHEMERAL, buffer, sizeof(buffer));
            // &ZOO_OPEN_ACL_UNSAFE, 0, buffer, sizeof(buffer));
    std::cout << "res[" << res << "]:"<< zerror(res) << std::endl;
    std::cout << buffer << std::endl;
}

int main() {
    zhandle_t *handle = init_zhandle();
    assert(handle != NULL);
    wait_until_connected();

    wexists(handle, "/ephemerals_node");//使用zoo_wexists检查是否存在
    exists(handle, "/ephemerals_node");//使用zoo_exists检查是否存在
    create(handle, "/ephemerals_node");//创建znode

    sleep(10);

    close_zhandle(handle);
    return 0;
}
```

部分代码与上个例子相同，因此做了删减。注意例子里context会有内存泄露的问题。

程序先使用两种同步方式检测/ephemerals_node这个znode是否存在，并注册监视点，接着创建这个结点，sleep等待回调函数被调用（真实场景里永远不建议使用sleep实现同步效果）。

从程序的输出看下`zoo_exist/zoo_wexists`的不同（过滤了zookeeper自身的输出并做了注释）：

```
zhandle_t* init_zhandle()//初始化连接
void zk_event_callback(zhandle_t*, int, int, const char*, void*)//连接成功回调
zh:0x619570
type:-1//ZOO_SESSION_EVENT
state:3//ZOO_CONNECTED_STATE
path://连接成功回调，没有监视path，因此为NULL
watcherCtx:0x617080
watcherCtx:zhandle_t* init_zhandle()//上下文
void wexists(zhandle_t*, const char*)//zoo_wexists注册watch_event_callback回调
res[-101]:no node//ZNONODE
void exists(zhandle_t*, const char*)//zoo_exists通过传入watch flag = 1注册回调
res[-101]:no node//ZNONODE
void create(zhandle_t*, const char*)//创建节点
void watch_event_callback(zhandle_t*, int, int, const char*, void*)//创建后触发zoo_wexists回调
zh:0x619570
type:1//ZOO_CREATED_EVENT
state:3//ZOO_CONNECTED_STATE
path:/ephemerals_node //监视path
watcherCtx:0x61a960
watcherCtx:void wexists(zhandle_t*, const char*)//上下文
void zk_event_callback(zhandle_t*, int, int, const char*, void*)//创建后触发zoo_exists回调
zh:0x619570
type:1//ZOO_CREATED_EVENT
state:3//ZOO_CONNECTED_STATE
path:/ephemerals_node //监视path
watcherCtx:0x61ab90
watcherCtx:void exists(zhandle_t*, const char*) //上下文
res[0]:ok //ZOK
/ephemerals_node
void close_zhandle(zhandle_t*&)
0   ok //ZOK
```

我们使用`zoo_exists/zoo_wexists`创建了两个监视点，`zoo_create`创建znode后，触发了两次回调，分别为`watch_event_callback zk_event_callback`。可以看到两个回调函数的context互不影响。

### 4. 写

上面的例子里，我们使用`zoo_create`创建了一个临时znode，函数原型如下：

```cpp
ZOOAPI int zoo_create(zhandle_t *zh, const char *path, const char *value,
        int valuelen, const struct ACL_vector *acl, int flags,
        char *path_buffer, int path_buffer_len);
```

ACL与权限管理有关，上个例子里不设置权限，因此取值为`ZOO_OPEN_ACL_UNSAFE`，ACL_vector初始化操作定义在zookeeper.jute.h。

之前我们介绍过znode一共有四种类型，其中参数flags用于指定类型：`ZOO_EPHEMERAL, ZOO_SEQUENCE, 0`，因为对sequence类型，创建的真实path在服务器端确定，参数`path_buffer`记录了真实的path。

创建后我们可以使用`zoo_set/zoo_set2`接口进行数据的更新

```cpp
ZOOAPI int zoo_set(zhandle_t *zh, const char *path, const char *buffer,
                   int buflen, int version);
ZOOAPI int zoo_set2(zhandle_t *zh, const char *path, const char *buffer,
                   int buflen, int version, struct Stat *stat);
```

参数`version`用于多个进程对同一znode数据的更新控制，只有传入的参数`version`与服务器端一致时，才会更新成功。

### 5. 读

读znode主要有三个接口：  
`zoo_exists/zoo_wexists`查看znode存在与否  
`zoo_get/zoo_wget`获取znode对应data  
`zoo_get_children/zoo_wget_children`获取子节点的信息  

各个接口的参数不再一一在本文赘述，注意一些参数初始化/销毁相关的操作在`zookeeper.jute.h`中定义。

### 6. 异步接口

异步接口跟同步类似，只是函数立刻返回，对znode的操作结果在异步函数里完成。例如`zoo_acreate`定义如下：

```cpp
ZOOAPI int zoo_acreate(zhandle_t *zh, const char *path, const char *value, 
        int valuelen, const struct ACL_vector *acl, int flags,
        string_completion_t completion, const void *data);
```

可以看到跟`zoo_create`最大的区别是多了一个异步结果处理函数以及对应的上下文。

其中`string_completion_t`的定义如下

```cpp
typedef void
        (*string_completion_t)(int rc, const char *value, const void *data);
```

`rc`为处理结果返回值，`value`为实际的path，`data`为`zoo_acreate`里传递的上下文对象，创建成功后该函数会被调用。

其他几个异步结果处理函数有：

```cpp
typedef void (*void_completion_t)(int rc, const void *data);
typedef void (*stat_completion_t)(int rc, const struct Stat *stat,
        const void *data);
typedef void (*data_completion_t)(int rc, const char *value, int value_len,
        const struct Stat *stat, const void *data);
typedef void (*strings_completion_t)(int rc,
        const struct String_vector *strings, const void *data);
typedef void (*strings_stat_completion_t)(int rc,
        const struct String_vector *strings, const struct Stat *stat,
        const void *data);
typedef void
        (*string_completion_t)(int rc, const char *value, const void *data);
typedef void (*acl_completion_t)(int rc, struct ACL_vector *acl,
        struct Stat *stat, const void *data);
```

### 7. 实战

之前介绍过使用`zlcli`模拟一个[主-从实例](http://izualzhy.cn/zkcli-example)，如果需要查看对应的master c代码，可以参考fpj大神的的[github](https://github.com/fpj/zookeeper-book-example)。

### 8. 注意

各个接口的用法建议在使用时直接参考`zookeeper.h`，因此本文没有逐个接口细细展开介绍，这里说明几个注意点：

1. zk的回调方式为单次回调，如果需要继续关注，需要在回调函数里再次注册监视点。  
2. 与zkcli相同，创建节点无法自动递归创建，需要手动创建。  
3. 监视点设置后无法手动移除。  
4. `zookeeper_interest`接口用于单线程版本里，在多线程环境下不建议使用。  
5. 单次触发会丢失事件，但是可以保证不丢任务。  
6. java中提供了`multitop`的调用方式保证多个操作的原子性，c提供了`zoo_multi`。  

### 9. 参考资料

1. [ZooKeeper：分布式过程协同技术详解](http://www.duokan.com/book/106575)
