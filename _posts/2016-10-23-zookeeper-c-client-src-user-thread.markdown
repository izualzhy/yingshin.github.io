---
title: zookeeper c客户端源码分析二：接口调用后发生了什么？
date: 2016-10-23 11:14:42
excerpt: "zookeeper c客户端源码分析二：接口调用后发生了什么？"
tags: zookeeper
---

本文介绍zk-clib暴露出来的接口调用后的处理流程，尝试解决以下几个疑问：

1. `zookeeper_init`返回后为什么不能确定连接已经建立？  
2. 同步接口和异步接口，例如`zoo_get/zoo_wget/zoo_aget/zoo_awget`调用后有什么区别？  

篇幅原因所有示例代码只摘抄了部分。

<!--more-->

### 1. zookeeper_init

作为第一个被调用的接口，从`zookeeper_init`开始入手

```
//构造并初始化各个成员变量，包括服务端的addr:port等，调用adaptor_init开启线程。
zhandle_t *zookeeper_init(const char *host, watcher_fn watcher,
  int recv_timeout, const clientid_t *clientid, void *context, int flags)
{
    log_env();
    LOG_INFO(("Initiating client connection, host=%s sessionTimeout=%d watcher=%p"
    zh->active_node_watchers=create_zk_hashtable();//hashtable创建
    zh->active_exist_watchers=create_zk_hashtable();
    zh->active_child_watchers=create_zk_hashtable();

    if (adaptor_init(zh) == -1) {
```

继续看`adaptor_init`，设置`self_pipe`后调用`start_threads`

```
    struct adaptor_threads *adaptor_threads = calloc(1, sizeof(*adaptor_threads));
    if (!adaptor_threads) {
        LOG_ERROR(("Out of memory"));
        return -1;
    }

    if(pipe(adaptor_threads->self_pipe)==-1) {
        LOG_ERROR(("Can't make a pipe %d",errno));
        free(adaptor_threads);
        return -1;
    }
    set_nonblock(adaptor_threads->self_pipe[1]);
    set_nonblock(adaptor_threads->self_pipe[0]);

    pthread_mutex_init(&zh->auth_h.lock,0);
    zh->adaptor_priv = adaptor_threads;
    start_threads(zh);
```

`self_pipe`用于程序结束时调用唤醒io线程

```
int wakeup_io_thread(zhandle_t *zh)
{
    struct adaptor_threads *adaptor_threads = zh->adaptor_priv;
    char c=0;
    return send(adaptor_threads->self_pipe[1], &c, 1, 0)==1? ZOK: ZSYSTEMERROR;
}
```

`start_threads`则启动了开启两个线程分别用于socket的读写和结果的处理。：

```
    rc=pthread_create(&adaptor->io, 0, do_io, zh);
    assert("pthread_create() failed for the IO thread"&&!rc);
    rc=pthread_create(&adaptor->completion, 0, do_completion, zh);
    assert("pthread_create() failed for the completion thread"&&!rc);
```

可以看到一次`zookeeper_init`启动了两个线程，入口函数分别为`do_io do_completion`。

这两个线程的作用我们留到下篇文章介绍，再看下其他接口例如`zoo_get`调用后发生了什么？

### 2. zoo_get

`zoo_get/zoo_wget/zoo_aget`实际上最终都是通过`zoo_awget`实现的。

`zoo_get`里调用了`zoo_wget`，如果要watch则传入`zh->watcher zh->context`，这两个变量在`zookeeper_init`里传入和初始化：

```
int zoo_get(zhandle_t *zh, const char *path, int watch, char *buffer,
        int* buffer_len, struct Stat *stat)
{
    return zoo_wget(zh,path,watch?zh->watcher:0,zh->context,
            buffer,buffer_len,stat);
}
```

`zoo_wget/zoo_aget`的实现都是调用了`zoo_awget`

```
int zoo_aget(zhandle_t *zh, const char *path, int watch, data_completion_t dc,
        const void *data)
{
    return zoo_awget(zh,path,watch?zh->watcher:0,zh->context,dc,data);
}

int zoo_wget(zhandle_t *zh, const char *path,
        watcher_fn watcher, void* watcherCtx,
        char *buffer, int* buffer_len, struct Stat *stat)
{
    struct sync_completion *sc;
    
    rc=zoo_awget(zh, path, watcher, watcherCtx, SYNCHRONOUS_MARKER, sc);
    if(rc==ZOK){
        wait_sync_completion(sc);
```

可以看到`zoo_wget`里使用`wait_sync_completion`等待signal。传入的回调函数是`SYNCHRONOUS_MARKER`，这个变量的定义比较有意思：

```
static void *SYNCHRONOUS_MARKER = (void*)&SYNCHRONOUS_MARKER;
```

`do_io`线程里会调用`notify_sync_completion`通知请求数据已经响应。

那么接下来，就到了真正的重点`zoo_awget`

```
int zoo_awget(zhandle_t *zh, const char *path,
        watcher_fn watcher, void* watcherCtx,
        data_completion_t dc, const void *data)
{
    struct oarchive *oa;
    char *server_path = prepend_string(zh, path);
    //构造请求header
    struct RequestHeader h = { STRUCT_INITIALIZER (xid , get_xid()), STRUCT_INITIALIZER (type ,ZOO_GETDATA_OP)};
    //构造请求buffer
    struct GetDataRequest req =  { (char*)server_path, watcher!=0 };
    int rc;

    if (zh==0 || !isValidPath(server_path, 0)) {
        free_duplicate_path(server_path, path);
        return ZBADARGUMENTS;
    }
    if (is_unrecoverable(zh)) {
        free_duplicate_path(server_path, path);
        return ZINVALIDSTATE;
    }
    oa=create_buffer_oarchive();
    //序列化
    rc = serialize_RequestHeader(oa, "header", &h);
    rc = rc < 0 ? rc : serialize_GetDataRequest(oa, "req", &req);
    enter_critical(zh);
    //构造completion_list_t并添加到zh->sent_requests
    rc = rc < 0 ? rc : add_data_completion(zh, h.xid, dc, data,
        create_watcher_registration(server_path,data_result_checker,watcher,watcherCtx));
    //添加到zh->to_send
    rc = rc < 0 ? rc : queue_buffer_bytes(&zh->to_send, get_buffer(oa),
            get_buffer_len(oa));
    leave_critical(zh);
    free_duplicate_path(server_path, path);
    /* We queued the buffer, so don't free it */
    close_buffer_oarchive(&oa, 0);

    LOG_DEBUG(("Sending request xid=%#x for path [%s] to %s",h.xid,path,
            format_current_endpoint_info(zh)));
    /* make a best (non-blocking) effort to send the requests asap */
    //尝试发送zh->to_send里的所有请求
    adaptor_send_queue(zh, 0);
    return (rc < 0)?ZMARSHALLINGERROR:ZOK;
}
```

从`zoo_awget`的实现可以总结出几点：

1. 请求被保存在了`zh->sent_requests`  
2. 请求序列化后保存在了`zh->to_send`，作为待发送的数据  
3. 接口设置的`watcher`并没有立刻被保存在`zh`几个相关的`zh_hashtable`里  

那么`watcher`何时更新到`zh_hashtable`里？`sent_requests`有什么作用？socket既然在`zookeeper_init`设置为nonblock的，那么`adaptor_send_queue`就有可能失败，`to_send`的数据是如何保证发送出去的？又是如何接收的？

我们留到下篇文章继续分解。





