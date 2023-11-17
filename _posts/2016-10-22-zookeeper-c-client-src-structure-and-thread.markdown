---
title: zookeeper c客户端源码分析一：数据结构与线程
date: 2016-10-22 21:32:49
excerpt: "zookeeper c客户端源码分析一：数据结构与线程"
tags: zookeeper
---

之前的几篇文章介绍了zookeeper的使用。周末稍微看了下zookeeper c客户端的代码，还是比较有意思的。虽然用纯C写成，实现复杂度会高一些，但是丝毫不影响可读性，代码量也不大。（我都不好意思说升级一个接手模块支持gcc4编译踩了无数个坑o(╯□╰)o）

想了解下下zookeeper c客户端主要好奇这么几个问题：  
1. zookeeper以回调的方式处理用户注册函数，那么是如何管理线程的？注册的回调函数、异步接口的返回数据回调是在同一个线程调用？应用方是否需要处理可能的通知乱序？  
2. 单次通知的方式是如何实现的？  
3. 注册znodeA的回调需要同步到服务端，在服务端返回确认注册回调前znodeA发生变化的话，该回调是否被调用？  
4. 传入多个服务集群地址时是如何选择连接哪台机器？一个还是多个？  
5. client与服务端的数据是如何发送/接收、序列化/反序列化的？  
6. 各种state:NOT_CONNECTED CONNECTING CONNECTED具体是怎么样变化的？  

学习zookeeper源码，可以看到线程的管理方式，链表/哈希表的使用，回调函数的管理， select/poll的使用等，包括各种平时用的越来越少`pipe/strdup`等。

本文首先介绍下zookeeper c客户端的线程模型和重要的数据结构。

<!--more-->

### 1. 数据结构

#### 1.1. zhandle

提起c客户端，最重要的数据结构莫过于`zhandle`

该结构体定义在`zk_adaptor.h`：

```
struct _zhandle {
#ifdef WIN32
    SOCKET fd; /* the descriptor used to talk to zookeeper */
#else
    int fd; /* the descriptor used to talk to zookeeper */
#endif
    char *hostname; /* the hostname of zookeeper */
    struct sockaddr_storage *addrs; /* the addresses that correspond to the hostname */
    int addrs_count; /* The number of addresses in the addrs array */
    watcher_fn watcher; /* the registered watcher */
    buffer_list_t *input_buffer; /* the current buffer being read in */
    buffer_head_t to_process; /* The buffers that have been read and are ready to be processed. */
    buffer_head_t to_send; /* The packets queued to send */
    completion_head_t sent_requests; /* The outstanding requests */
    completion_head_t completions_to_process; /* completions that are ready to run */
    int connect_index; /* The index of the address to connect to */
    clientid_t client_id;
    long long last_zxid;
    int outstanding_sync; /* Number of outstanding synchronous requests */
    struct _buffer_list primer_buffer; /* The buffer used for the handshake at the start of a connection */
    struct prime_struct primer_storage; /* the connect response */
    char primer_storage_buffer[40]; /* the true size of primer_storage */
    volatile int state;
    void *context;
    volatile int close_requested;

    zk_hashtable* active_node_watchers;
    zk_hashtable* active_exist_watchers;
    zk_hashtable* active_child_watchers;
    /** used for chroot path at the client side **/
    char *chroot;
};
```

篇幅原因，省略掉了部分成员变量。  
部分成员变量直接使用`zookeeper_init`传入的参数赋值，例如`hostname watcher client_id context`，有的则是计算参数得到，例如`addrs`是从`hostname`里解析出的所有地址端口，`addrs_count`则是服务端的机器数目。

连接的机器从`addrs`选取，`connect_index`则是对应的下标。其中建立连接部分在`zookeeper_interest`实现，我们将在下篇文章介绍。

接下来我们从这个结构体出发，介绍下基本的数据结构、线程模型以及socket读写。

#### 1.2. zh_hashtable

`zhandle`类型里

```
    zk_hashtable* active_node_watchers;
    zk_hashtable* active_exist_watchers;
    zk_hashtable* active_child_watchers;
```

存储我们注册的所有watchers。

`zk_hashtable`就是哈希表，发布在`src/hashtable`下。**可以单独拎出来编译使用**。

注意`zk_hashtable`使用的hash函数和比较函数分别为:

```
static unsigned int string_hash_djb2(void *str) 
{
    unsigned int hash = 5381;
    int c;
    const char* cstr = (const char*)str;
    while ((c = *cstr++))
        hash = ((hash << 5) + hash) + c; /* hash * 33 + c */

    return hash;
}

static int string_equal(void *key1,void *key2)
{
    return strcmp((const char*)key1,(const char*)key2)==0;
}
```

hashtable.h中提供了hash表的常用接口：`create/insert/search/remove/count/destroy`  
hashtable_itr.h则实现了hash表的迭代器：`/advance/remove/search`  

例如单独使用hastable下的源码：  

```
    struct hashtable* h = create_hashtable(32, string_hash_djb2, string_equal);
    for (int i = 0; i < 10; ++i) {
        int* k = (int*)malloc(sizeof(int));
        *k = i;
        char* v = (char*)malloc(sizeof(char));
        *v = 'a' + i;
        hashtable_insert(h, k, v);
    }

    struct hashtable_itr* itr = hashtable_iterator(h);
    while (true) {
        void* k = hashtable_iterator_key(itr);
        void* v = hashtable_iterator_value(itr);
        printf("k:%d v:%c\n", *(int*)(k), *(char*)v);
        if (hashtable_iterator_advance(itr) == 0) {
            break;
        }
    }
```

zookeeper源码里hashtable为`zk_hashtable`，key为`const char*`，使用时就是上层需要监视的znode-path。

value为`watcher_object_t*`类型。

```
typedef struct _watcher_object {
    watcher_fn watcher;
    void* context;
    struct _watcher_object* next;
} watcher_object_t;

struct watcher_object_list {
    watcher_object_t* head;
};
```

`watcher_fn`在之前的介绍中已经很熟悉了，就是我们的监视点回调函数  
`context`则为对应的上下文  
`next`使得监视点以链表的形式存储下来  

实际使用时，从对应的hash表中取出对应的所有监视点，以`watcher_object_list`的形式返回给调用方。  

#### 1.3. 链表

c 客户端源码中使用了大量的链表，比如上节中的`watcher_object_list`。

更加典型的是

```
    buffer_list_t *input_buffer; /* the current buffer being read in */
    buffer_head_t to_process; /* The buffers that have been read and are ready to be processed. */
    buffer_head_t to_send; /* The packets queued to send */
```

先提前介绍下各变量的用途：`input_buffer`实际上一直都作为一个链表Node使用，从服务端接收的数据存储在这里。接收一个完整的数据包后添加到`to_process`链表。zookeeper各个接口例如`zoo_get`的请求数据都放到`to_send`链表，包括ping的数据包请求。

其中`buffer_list_t`是链表的node定义：

```
typedef struct _buffer_list {
    char *buffer;//数据
    int len; /* This represents the length of sizeof(header) + length of buffer */需要读取的(header + buffer)数据长度
    int curr_offset; /* This is the offset into the header followed by offset into the buffer */(数据包长度 + header + buffer)已经读取到的数据总长度
    struct _buffer_list *next;
} buffer_list_t;
```

`buffer_head_t`定义了链表的头尾：

```
typedef struct _buffer_head {
    struct _buffer_list *volatile head;//链表第一个元素
    struct _buffer_list *last;//链表最后一个元素
#ifdef THREADED
    pthread_mutex_t lock;
#endif
} buffer_head_t;
```

可以看到除了定义链表的头尾元素，还定义了`pthread_mutex_t lock`，基于此提供了两个接口对链表加锁和解锁，实际上就是调用了`pthread_mutex_lock/unlock`的原语：

```
int lock_buffer_list(buffer_head_t *l)
{
    return pthread_mutex_lock(&l->lock);
}
int  unlock_buffer_list(buffer_head_t *l)
{
    return pthread_mutex_unlock(&l->lock);
}
```

当然`zookeeper.c`里实现了链表的一系列操作，实现对链表的CRUD操作：

```
//申请一块buffer_list_t内存并初始化，成员变量buffer使用传入的buff
static buffer_list_t *allocate_buffer(char *buff, int len)

//清空b b->buffer占用的内存
static void free_buffer(buffer_list_t *b)

//删除list首部的元素，如果删除后list为空，则list->head = list->last = NULL
static buffer_list_t *dequeue_buffer(buffer_head_t *list)

//从list删除首部元素并释放内存，实际上就是调用了dequeue_buffer和free_buffer
static int remove_buffer(buffer_head_t *list)

//添加list到链表b，如果add_to_front为真，添加到链表首部。如果链表只有1个元素，那么list->head = list->last = b
static void queue_buffer(buffer_head_t *list, buffer_list_t *b, int add_to_front)

//使用buff len构造buffer_list_t并add到list尾部
static int queue_buffer_bytes(buffer_head_t *list, char *buff, int len)

//使用buff len构造buffer_list_t并add到list首部
static int queue_front_buffer_bytes(buffer_head_t *list, char *buff, int len)
```

#### 1.4. completion_list_t

`zhandle`里还有一种链表`completion_head_t`

```
    completion_head_t sent_requests; /* The outstanding requests */
    completion_head_t completions_to_process; /* completions that are ready to run */
```

其中Node类型为`completion_list_t`

```
typedef struct _completion_head {
    struct _completion_list *volatile head;
    struct _completion_list *last;
#ifdef THREADED
    pthread_cond_t cond;
    pthread_mutex_t lock;
#endif
} completion_head_t;
```

可以看到比起`_buffer_head`多了一个条件变量。因此加锁/解锁的封装也略有不同：

```
int lock_completion_list(completion_head_t *l)
{
    return pthread_mutex_lock(&l->lock);
}
int unlock_completion_list(completion_head_t *l)
{
    pthread_cond_broadcast(&l->cond);
    return pthread_mutex_unlock(&l->lock);
}
```

接着看链表的Node类型`_completion_list`：

```
typedef struct _completion_list {
    int xid;
    completion_t c;
    const void *data;
    buffer_list_t *buffer;
    struct _completion_list *next;
    watcher_registration_t* watcher;
} completion_list_t;
```

#### 1.5 sync_completion

同步接口调用时，`sync_completion`将异步接口使用`wait/signal`的方式起到同步的效果。

```
struct sync_completion {
    int rc;
    union {
        struct {
            char *str;
            int str_len;
        } str;
        struct Stat stat;
        struct {
            char *buffer;
            int buff_len;
            struct Stat stat;
        } data;
        struct {
            struct ACL_vector acl;
            struct Stat stat;
        } acl;
        struct String_vector strs2;
        struct {
            struct String_vector strs2;
            struct Stat stat2;
        } strs_stat;
    } u;
    int complete;
#ifdef THREADED
    pthread_cond_t cond;
    pthread_mutex_t lock;
#endif
};
```

`u`封装了数据本身，`wait/signal`实现方式如下：

```
int wait_sync_completion(struct sync_completion *sc)
{
    pthread_mutex_lock(&sc->lock);
    while (!sc->complete) {
        pthread_cond_wait(&sc->cond, &sc->lock);
    }
    pthread_mutex_unlock(&sc->lock);
    return 0;
}
void notify_sync_completion(struct sync_completion *sc)
{
    pthread_mutex_lock(&sc->lock);
    sc->complete = 1;
    pthread_cond_broadcast(&sc->cond); 
    pthread_mutex_unlock(&sc->lock);
}
```

### 2. 线程模型

线程相关数据存储在

```
/* this is used by mt_adaptor internally for thread management */
struct adaptor_threads {
     pthread_t io; 
     pthread_t completion;
     int threadsToWait;         // barrier
     pthread_cond_t cond;       // barrier's conditional
     pthread_mutex_t lock;      // ... and a lock
     pthread_mutex_t zh_lock;   // critical section lock
#ifdef WIN32
     SOCKET self_pipe[2];
#else
     int self_pipe[2];
#endif
};
```

实际上`zhandle_t`里的`void *adaptor_priv`就是该类型，在`adaptor_init`里初始化。

两个线程对应的入口函数分别为：`do_io do_completion`。

### 3. socket读写

zk-clib封装了`recv_buffer send_buffer`接收和发送数据。

其中协议为前4个字节表示buffer长度，之后为具体的buffer。

```
/* returns:
 * -1 if recv call failed,
 * 0 if recv would block,
 * 1 if success
 */
//从fd接收数据，前4个字节记录存储buffer长度
//接收后存储到buffer->len，接收到的数据存储到buffer->buffer
//buffer->curr_offset存储了已经接收到的所有数据长度
static int recv_buffer(int fd, buffer_list_t *buff)
/* returns:
 * -1 if send failed,
 * 0 if send would block while sending the buffer (or a send was incomplete),
 * 1 if success
 */
static int send_buffer(int fd, buffer_list_t *buff)
```

socket的读写有两种：

1. 建立连接后的第一次信息交换：协议版本/zxid/passwd等数据  
2. 之后的请求种类比较多样：接口请求/ping数据/连接请求/重连后的set watcher请求等。

### 4. 序列化/反序列化

#### 4.1 connect

`zhandle`里以下三个成员变量用于跟服务端的第一次信息交换：

```
    struct _buffer_list primer_buffer; /* The buffer used for the handshake at the start of a connection */
    struct prime_struct primer_storage; /* the connect response */
    char primer_storage_buffer[40]; /* the true size of primer_storage */
```

其中`primer_buffer`存储了客户端的第一次连接请求，通过`connect_req`序列化后填充。

```
struct connect_req {
    int32_t protocolVersion;
    int64_t lastZxidSeen;
    int32_t timeOut;
    int64_t sessionId;
    int32_t passwd_len;
    char passwd[16];
};
```

返回的结果存储在`primer_storage_buffer`，这是一个40字节的数组，这个数字跟`primer_storage`结构体大小对应，接收数据存储在buffer后反序列化填充`primer_storage`。

```
/* connect request */
/* the connect response */
struct prime_struct {
    int32_t len;
    int32_t protocolVersion;
    int32_t timeOut;
    int64_t sessionId;
    int32_t passwd_len;
    char passwd[16];
};
```

以上序列化和反序列化的实现：

```
static int serialize_prime_connect(struct connect_req *req, char* buffer)
static int deserialize_prime_response(struct prime_struct *req, char* buffer)
```

其实就是按照成员变量定义的顺序逐个字节写到buffer，或者从buffer解析到成员变量。


#### 4.2 after-connected

信息交换后各种请求的序列化过程，以`set_get`为例摘录了重要的相关代码：

```
    //STRUCT_INITIALIZER是为了跨平台定义的MACRO
    struct RequestHeader h = { STRUCT_INITIALIZER (xid , get_xid()), STRUCT_INITIALIZER (type ,ZOO_GETDATA_OP)};
    struct GetDataRequest req =  { (char*)server_path, watcher!=0 };
    ...
    oa=create_buffer_oarchive();
    rc = serialize_RequestHeader(oa, "header", &h); 
    rc = rc < 0 ? rc : serialize_GetDataRequest(oa, "req", &req);
    enter_critical(zh);
    rc = rc < 0 ? rc : add_data_completion(zh, h.xid, dc, data,
        create_watcher_registration(server_path,data_result_checker,watcher,watcherCtx));
    rc = rc < 0 ? rc : queue_buffer_bytes(&zh->to_send, get_buffer(oa),
            get_buffer_len(oa));
```

可以看到buffer分成了`RequestHeader GetDataRequest`两部分，所有类型的请求里header都是`RequestHeader`类型，data则根据不同的请求类型不同，header里的`xid type`描述了data的类型。

```
struct RequestHeader {
    int32_t xid;
    int32_t type;
};
```

对于普通的znode操作请求，xid是一个递增的值，通过`get_xid`获取。

可选的其他值还有：

```
/* predefined xid's values recognized as special by the server */
#define WATCHER_EVENT_XID -1 
#define PING_XID -2
#define AUTH_XID -4
#define SET_WATCHES_XID -8
```

type描述了不同的请求，例如对于`get`请求，`type = ZOO_GETDATA_OP`。其他定义在`proto.h`里：

```
#define ZOO_NOTIFY_OP 0
#define ZOO_CREATE_OP 1
#define ZOO_DELETE_OP 2
#define ZOO_EXISTS_OP 3
#define ZOO_GETDATA_OP 4
#define ZOO_SETDATA_OP 5
#define ZOO_GETACL_OP 6
#define ZOO_SETACL_OP 7
#define ZOO_GETCHILDREN_OP 8
#define ZOO_SYNC_OP 9
#define ZOO_PING_OP 11
#define ZOO_GETCHILDREN2_OP 12
#define ZOO_CHECK_OP 13
#define ZOO_MULTI_OP 14
#define ZOO_CLOSE_OP -11
#define ZOO_SETAUTH_OP 100
#define ZOO_SETWATCHES_OP 101
```

序列化的接口通过`serialize_xxx`接口完成，类似于例子里的`serialize_RequestHeader serialize_GetDataRequest`。

反序列化的过程类似，摘录了部分代码：

```
        struct ReplyHeader hdr;
        buffer_list_t *bptr = cptr->buffer;
        struct iarchive *ia = create_buffer_iarchive(bptr->buffer,
                bptr->len);
        //首先填充hdr
        deserialize_ReplyHeader(ia, "hdr", &hdr);

        if (hdr.xid == WATCHER_EVENT_XID) {
            int type, state;
            struct WatcherEvent evt;
            //解析得到对应的type state path填充evt
            deserialize_WatcherEvent(ia, "event", &evt);
            ...
```

首先填充ReplyHeader，根据hdr.xid的不同调用`deserialize_xxx`接口。

其中`ReplyHeader`接口定义：

```
struct ReplyHeader {
    int32_t xid;
    int64_t zxid;
    int32_t err;
};
```

序列化的相关接口都隐藏在`zookeeper.jute.h`。

基本的数据结构、socket封装、序列化/反序列化先介绍这些。

下篇文章开始，我们开始介绍流程的处理过程，了解了数据结构、接口的调用后，可能更容易理解本文以及作者对于数据结构的设计意图。
