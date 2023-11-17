---
title: zookeeper c客户端源码分析三：两个线程
date: 2016-10-23 12:39:50
excerpt: "zookeeper c客户端源码分析三：两个线程"
tags: zookeeper
---

[上篇文章](http://izualzhy.cn/zookeeper-c-client-src-user-thread)介绍了`zookeeper_init`开启了两个线程，本文主要看下这两个线程`do_io do_completion`的流程。

<!--more-->

### 1. do_io

```
void *do_io(void *v)
{
    zhandle_t *zh = (zhandle_t*)v;
    fd_set rfds, wfds, efds;
    struct adaptor_threads *adaptor_threads = zh->adaptor_priv;
    api_prolog(zh);
    notify_thread_ready(zh);
    LOG_DEBUG(("started IO thread"));
    FD_ZERO(&rfds);   FD_ZERO(&wfds);    FD_ZERO(&efds);
    while(!zh->close_requested) {//初始化为0，zookeeper_close设置为1
        struct timeval tv;
        SOCKET fd;
        SOCKET maxfd=adaptor_threads->self_pipe[0];
        int interest;
        int rc;
       //zookeeper_interser负责连接服务集群，输出
       //fd:与服务端建立的连接描述符
       //interest:读写事件类型
       //tv:select等待时间
       zookeeper_interest(zh, &fd, &interest, &tv);
       if (fd != -1) {
           //可读->FS_SET 不可读->FD_CLR
           if (interest&ZOOKEEPER_READ) {
                FD_SET(fd, &rfds);
            } else {
                FD_CLR(fd, &rfds);
            }
           //可写->FS_SET 不可写->FD_CLR
           if (interest&ZOOKEEPER_WRITE) {
                FD_SET(fd, &wfds);
            } else {
                FD_CLR(fd, &wfds);
            }
        }
       //监听self_pipe[0]
       FD_SET( adaptor_threads->self_pipe[0] ,&rfds );
       //监听与zk服务集群server连接的fd, self_pipe[0]
       rc = select((int)maxfd, &rfds, &wfds, &efds, &tv);
       if (fd != -1) 
       {
           interest = (FD_ISSET(fd, &rfds))? ZOOKEEPER_READ:0;
           interest|= (FD_ISSET(fd, &wfds))? ZOOKEEPER_WRITE:0;
        }
       //self_pipe[0]可读
       if (FD_ISSET(adaptor_threads->self_pipe[0], &rfds)){
            // flush the pipe/socket
            char b[128];
           while(recv(adaptor_threads->self_pipe[0],b,sizeof(b), 0)==sizeof(b)){}
       }
        // dispatch zookeeper events
        rc = zookeeper_process(zh, interest);
        // check the current state of the zhandle and terminate 
        // if it is_unrecoverable()
        // ZOO_EXPIRED_SESSION_STATE or ZOO_AUTH_FAILED_STATE两种状态认为unrecoverable
        if(is_unrecoverable(zh))
            break;
    }
    api_epilog(zh, 0);    
    LOG_DEBUG(("IO thread terminated"));
    return 0;
}
```

`zookeeper_interest`使用round-robin选择服务端地址并连接，连接后调用`prime_connection`发起第一次数据交换，`interest`为可监视的事件类型(ZOOKEEPER_READ/ZOOKEEPER_WRITE)，`tv`为计算的超时时间。

`prime_connection`发送`protocolVersion/sessionId`等数据并修改`zh->state`为`ZOO_ASSOCIATING_STATE`

```
static int prime_connection(zhandle_t *zh)
{
    int rc;
    /*this is the size of buffer to serialize req into*/
    char buffer_req[HANDSHAKE_REQ_SIZE];
    int len = sizeof(buffer_req);
    int hlen = 0;
    struct connect_req req;
    req.protocolVersion = 0;
    req.sessionId = zh->client_id.client_id;
    req.passwd_len = sizeof(req.passwd);
    memcpy(req.passwd, zh->client_id.passwd, sizeof(zh->client_id.passwd));
    req.timeOut = zh->recv_timeout;
    req.lastZxidSeen = zh->last_zxid;
    hlen = htonl(len);
    /* We are running fast and loose here, but this string should fit in the initial buffer! */
    //首先发送包长度
    rc=zookeeper_send(zh->fd, &hlen, sizeof(len));
    //序列化req到buffer_req
    serialize_prime_connect(&req, buffer_req);
    //接着发送buffer_req
    rc=rc<0 ? rc : zookeeper_send(zh->fd, buffer_req, len);
    if (rc<0) {
        return handle_socket_error_msg(zh, __LINE__, ZCONNECTIONLOSS,
                "failed to send a handshake packet: %s", strerror(errno));
    }
    //修改state
    zh->state = ZOO_ASSOCIATING_STATE;
    //设置primer_buffer用于接收数据
    zh->input_buffer = &zh->primer_buffer;
    /* This seems a bit weird to to set the offset to 4, but we already have a
     * length, so we skip reading the length (and allocating the buffer) by
     * saying that we are already at offset 4 */
    zh->input_buffer->curr_offset = 4;

    return ZOK;
}
```

`zookeeper_interest`返回后，根据`interest`监视与服务端的接口`fd`的行为，同时监视`self_pipe[0]`的可读。之后调用`int zookeeper_process(zhandle_t *zh, int events)`继续处理。

`self_pipe[1]`的写操作在`wakeup_io_thread`实现，用于立刻唤醒io线程。

`zookeeper_process`首先使用`check_events`处理socket的读写数据

```
int zookeeper_process(zhandle_t *zh, int events)
{
    buffer_list_t *bptr;
    int rc;

    if (zh==NULL)
        return ZBADARGUMENTS;
    if (is_unrecoverable(zh))
        return ZINVALIDSTATE;
    api_prolog(zh);
    IF_DEBUG(checkResponseLatency(zh));
    rc = check_events(zh, events);
    if (rc!=ZOK)
        return api_epilog(zh, rc);
```

先看下`check_events`的实现：

```
//如果events是可写事件，则发送zh->to_send的所有数据
//如果events是可读事件，则接收。
//普通数据包加入到zh->to_process
//连接包解析后更新zh->primer_storage
static int check_events(zhandle_t *zh, int events)
{
    if (zh->fd == -1)
        return ZINVALIDSTATE;
    if ((events&ZOOKEEPER_WRITE)&&(zh->state == ZOO_CONNECTING_STATE)) {
        int rc, error;
        socklen_t len = sizeof(error);
        rc = getsockopt(zh->fd, SOL_SOCKET, SO_ERROR, &error, &len);
        /* the description in section 16.4 "Non-blocking connect"
         * in UNIX Network Programming vol 1, 3rd edition, points out
         * that sometimes the error is in errno and sometimes in error */
        if (rc < 0 || error) {
            if (rc == 0)
                errno = error;
            return handle_socket_error_msg(zh, __LINE__,ZCONNECTIONLOSS,
                "server refused to accept the client");
        }
        //发送连接数据到服务端，修改zh->state = ZOO_ASSOCIATING_STATE
        if((rc=prime_connection(zh))!=0)
            return rc;
        LOG_INFO(("initiated connection to server [%s]",
                format_endpoint_info(&zh->addrs[zh->connect_index])));
        return ZOK;
    }
    //如果有待发送的数据(to_send) 且 socket可写
    if (zh->to_send.head && (events&ZOOKEEPER_WRITE)) {
        /* make the flush call non-blocking by specifying a 0 timeout */
        //发送zh->to_send链表里的所有数据
        int rc=flush_send_queue(zh,0);
        if (rc < 0)
            return handle_socket_error_msg(zh,__LINE__,ZCONNECTIONLOSS,
                "failed while flushing send queue");
    }
    //socket可读
    if (events&ZOOKEEPER_READ) {
        int rc;
        //如果当前没有可用的接收buffer，申请一块可用buffer内存。
        //连接时,zh->input_buffer = &zh->prime_buffer
        if (zh->input_buffer == 0) {
            zh->input_buffer = allocate_buffer(0,0);
        }

        //接收数据到zh->input_buffer
        //handshake阶段，也就是input == &primer_buffer时，接收数据在zh->input_buffer->buffer[0]
        rc = recv_buffer(zh->fd, zh->input_buffer);
        if (rc < 0) {
            return handle_socket_error_msg(zh, __LINE__,ZCONNECTIONLOSS,
                "failed while receiving a server response");
        }
        if (rc > 0) {
            gettimeofday(&zh->last_recv, 0);
            //如果不是handshake的包，则加入到zh->to_process链表
            if (zh->input_buffer != &zh->primer_buffer) {
                queue_buffer(&zh->to_process, zh->input_buffer, 0);
            } else  {
                //第一次信息交换返回的response
                int64_t oldid,newid;
                //deserialize
                //反序列化接收到的数据(zh->primer_buffer.buffer)到zh->primer_storage
                deserialize_prime_response(&zh->primer_storage, zh->primer_buffer.buffer);
                /* We are processing the primer_buffer, so we need to finish
                 * the connection handshake */
                oldid = zh->client_id.client_id;
                newid = zh->primer_storage.sessionId;
                if (oldid != 0 && oldid != newid) {
                    zh->state = ZOO_EXPIRED_SESSION_STATE;
                    errno = ESTALE;
                    return handle_socket_error_msg(zh,__LINE__,ZSESSIONEXPIRED,
                            "sessionId=%#llx has expired.",oldid);
                } else {
                    zh->recv_timeout = zh->primer_storage.timeOut;
                    zh->client_id.client_id = newid;
                 
                    memcpy(zh->client_id.passwd, &zh->primer_storage.passwd,
                           sizeof(zh->client_id.passwd));
                    //修改state
                    zh->state = ZOO_CONNECTED_STATE;
                    LOG_INFO(("session establishment complete on server [%s], sessionId=%#llx, negotiated timeout=%d",
                              format_endpoint_info(&zh->addrs[zh->connect_index]),
                              newid, zh->recv_timeout));
                    /* we want the auth to be sent for, but since both call push to front
                       we need to call send_watch_set first */
                    //发送watch信息
                    send_set_watches(zh);
                    /* send the authentication packet now */
                    send_auth_info(zh);
                    LOG_DEBUG(("Calling a watcher for a ZOO_SESSION_EVENT and the state=ZOO_CONNECTED_STATE"));
                    zh->input_buffer = 0; // just in case the watcher calls zookeeper_process() again
                    PROCESS_SESSION_EVENT(zh, ZOO_CONNECTED_STATE);
                }
            }
            zh->input_buffer = 0;
        } else {
            // zookeeper_process was called but there was nothing to read
            // from the socket
            return ZNOTHING;
        }
    }
    return ZOK;
}
```

`check_events`根据不同的socket状态处理：  
1. 如果可写，则发送`zh->to_send`的所有数据  
2. 如果可读，则接收数据，判断是否是`prime_connection`发送的数据包，如果不是则解析后`zh->to_process`链表，如果是则解析后存储到`prime_storage`，更新本地信息并修改状态为`ZOO_CONNECTED_STATE`。`PROCESS_SESSIOIN_EVENT`构造connected事件添加到`completions_to_process`，之后经过`do_completion`处理，调用应用方设置的回调函数，状态为`ZOO_CONNECTED_STATE`。

继续看`zookeeper_process`的操作：

```
    //从to_process链表逐个取出接收到的buffer
    while (rc >= 0 && (bptr=dequeue_buffer(&zh->to_process))) {
        struct ReplyHeader hdr;
        struct iarchive *ia = create_buffer_iarchive(
                                    bptr->buffer, bptr->curr_offset);
        //解析hdr,hdr.xid表示数据类型
        deserialize_ReplyHeader(ia, "hdr", &hdr);
        if (hdr.zxid > 0) {
            zh->last_zxid = hdr.zxid;
        } else {
            // fprintf(stderr, "Got %#x for %#x\n", hdr.zxid, hdr.xid);
        }

        //判断解析出的hdr.xid
        if (hdr.xid == PING_XID) {
            // Ping replies can arrive out-of-order
            int elapsed = 0;
            struct timeval now;
            gettimeofday(&now, 0);
            elapsed = calculate_interval(&zh->last_ping, &now);
            LOG_DEBUG(("Got ping response in %d ms", elapsed));
            free_buffer(bptr);
        } else if (hdr.xid == WATCHER_EVENT_XID) {
            //watch事件通知
            struct WatcherEvent evt;
            int type = 0;
            char *path = NULL;
            completion_list_t *c = NULL;

            LOG_DEBUG(("Processing WATCHER_EVENT"));

            //解析数据到evt
            deserialize_WatcherEvent(ia, "event", &evt);
            type = evt.type;
            path = evt.path;
            /* We are doing a notification, so there is no pending request */
            c = create_completion_entry(WATCHER_EVENT_XID,-1,0,0,0,0);
            c->buffer = bptr;
            //根据type path查找相关的hashtable(zh->active_node/exist/child_watches)
            //move hashtable里对应的value到c->c.watcher_result(链表类型，如果有多个hashtable，则追加)
            c->c.watcher_result = collectWatchers(zh, type, path);

            // We cannot free until now, otherwise path will become invalid
            deallocate_WatcherEvent(&evt);
            //添加到zh->completions_to_process，交给do_completion线程处理
            queue_completion(&zh->completions_to_process, c, 0);
        } else if (hdr.xid == SET_WATCHES_XID) {
            LOG_DEBUG(("Processing SET_WATCHES"));
            free_buffer(bptr);
        } else if (hdr.xid == AUTH_XID){
            LOG_DEBUG(("Processing AUTH_XID"));

            /* special handling for the AUTH response as it may come back
             * out-of-band */
            auth_completion_func(hdr.err,zh);
            free_buffer(bptr);
            /* authentication completion may change the connection state to
             * unrecoverable */
            if(is_unrecoverable(zh)){
                handle_error(zh, ZAUTHFAILED);
                close_buffer_iarchive(&ia);
                return api_epilog(zh, ZAUTHFAILED);
            }
        } else {
            int rc = hdr.err;
            /* Find the request corresponding to the response */
            completion_list_t *cptr = dequeue_completion(&zh->sent_requests);

            /* [ZOOKEEPER-804] Don't assert if zookeeper_close has been called. */
            if (zh->close_requested == 1 && cptr == NULL) {
                LOG_DEBUG(("Completion queue has been cleared by zookeeper_close()"));
                close_buffer_iarchive(&ia);
                free_buffer(bptr);
                return api_epilog(zh,ZINVALIDSTATE);
            }
            assert(cptr);
            /* The requests are going to come back in order */
            if (cptr->xid != hdr.xid) {
                LOG_DEBUG(("Processing unexpected or out-of-order response!"));

                // received unexpected (or out-of-order) response
                close_buffer_iarchive(&ia);
                free_buffer(bptr);
                // put the completion back on the queue (so it gets properly
                // signaled and deallocated) and disconnect from the server
                queue_completion(&zh->sent_requests,cptr,1);
                return handle_socket_error_msg(zh, __LINE__,ZRUNTIMEINCONSISTENCY,
                        "unexpected server response: expected %#x, but received %#x",
                        hdr.xid,cptr->xid);
            }

            //添加到对应zk_hashtable:active_node_watches/active_exist_watches/active_child_watches
            activateWatcher(zh, cptr->watcher, rc);

            //异步调用则将结果插入zh->completions_to_process由do_completion线程处理
            if (cptr->c.void_result != SYNCHRONOUS_MARKER) {
                LOG_DEBUG(("Queueing asynchronous response"));
                cptr->buffer = bptr;
                queue_completion(&zh->completions_to_process, cptr, 0);
            } else {
                struct sync_completion
                        *sc = (struct sync_completion*)cptr->data;
                sc->rc = rc;
                
                process_sync_completion(cptr, sc, ia, zh); 
                //zookeeper的同步接口会调用wait_sync_completion等待notify
                notify_sync_completion(sc);
                free_buffer(bptr);
                zh->outstanding_sync--;
                destroy_completion_entry(cptr);
            }
        }

        close_buffer_iarchive(&ia);

    }
```

逐个处理`zh->to_process`链表的数据，首先解析出`ReplyHeader`，根据xid不同的处理：

1. PING_XID:`send_ping`发送的数据返回，更新`zh->last_ping`  
2. WATCHER_EVENT_XID:watcher节点的通知，`collectWatchers`根据不同的事件类型，znode路径从`active_node_watchers/active_exist_watchers/active_child_watchers`选择需要调用的回调，注意这里是从各个hashtable里remove的操作，因此实现了单次回调，添加到`completions_to_process`链表。  
3. SET_WATCHES_XID:`send_set_watches`发送的数据返回。  
4. AUTH_XID：acl设置返回数据。  

根据之前的介绍，正常的数据请求时`RequestHeader.xid`被设置成一个递增的整数，也就是`else`后的处理逻辑。

这些返回数据需要和`zh->sent_requests`一一对应，因此从链表头部取出数据。如果`xid`不能一一对应的话，说明有一些丢包、乱序的情况，此时把取出的数据重新放回到链表头部，也就是为什么链表有一个添加数据到首部的接口设计。

同时，只有接收到服务端的返回包后，监视点的回调函数才被添加到对应的`zk_hashtable`里。接下来根据`SYNCHRONOUS_MARKER`判断是否是同步接口，如果是同步接口，则调用`notify_sync_completion`通知同步接口调用线程，如果是异步接口，则添加到`completions_to_process`链表。

### 2. do_completion

`do_completion`等待`completions_to_process`添加数据，调用`process_completions`处理

```
void *do_completion(void *v)
{
    zhandle_t *zh = v;
    api_prolog(zh);
    notify_thread_ready(zh);
    LOG_DEBUG(("started completion thread"));
    while(!zh->close_requested) {
        pthread_mutex_lock(&zh->completions_to_process.lock);
        //等待zh->completions_to_process链表里添加数据
        while(!zh->completions_to_process.head && !zh->close_requested) {
            pthread_cond_wait(&zh->completions_to_process.cond, &zh->completions_to_process.lock);
        }
        pthread_mutex_unlock(&zh->completions_to_process.lock);
        process_completions(zh);
    }
    api_epilog(zh, 0);    
    LOG_DEBUG(("completion thread terminated"));
    return 0;
}
```

`process_completions`从`zh->completioins_to_process`取出数据，如果是`WATCHER_EVENT_XID`并调用响应的watcher处理，如果是异步的结果处理则在`deserialize_response`处理。

```
/* handles async completion (both single- and multithreaded) */
void process_completions(zhandle_t *zh)
{
    completion_list_t *cptr;
    //从zh->completions_to_process里取出数据
    while ((cptr = dequeue_completion(&zh->completions_to_process)) != 0) {
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
            /* We are doing a notification, so there is no pending request */
            type = evt.type;
            state = evt.state;
            /* This is a notification so there aren't any pending requests */
            LOG_DEBUG(("Calling a watcher for node [%s], type = %d event=%s",
                       (evt.path==NULL?"NULL":evt.path), cptr->c.type,
                       watcherEvent2String(type)));
            //依次调用cptr->c.watcher_result下的回调
            deliverWatchers(zh,type,state,evt.path, &cptr->c.watcher_result);
            deallocate_WatcherEvent(&evt);
        } else {
            deserialize_response(cptr->c.type, hdr.xid, hdr.err != 0, hdr.err, cptr, ia);
        }
        destroy_completion_entry(cptr);
        close_buffer_iarchive(&ia);
    }
}
```
