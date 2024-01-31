---
title: 使用curl进行http高并发访问
date: 2016-06-01 18:30:21
excerpt: "使用curl进行http高并发访问"
tags: cpp
---

本文主要介绍curl异步接口的使用方式，以及获取高性能的一些思路和实践。同时假设读者已经熟悉并且使用过同步接口。

<!--more-->

#### 1.curl接口基本介绍

curl一共有三种接口：  
* [Easy Interface](https://curl.haxx.se/libcurl/c/libcurl-easy.html)  
* [Multi Interface](https://curl.haxx.se/libcurl/c/libcurl-multi.html)  
* [Share Interface](https://curl.haxx.se/libcurl/c/libcurl-share.html)  

##### 1.1 Easy Interface

Easy下是同步接口，curl_easy_*的形式，基本处理方式有几个步骤：  
1. curl_easy_init获取easy handle  
2. curl_easy_setopt设置header/cookie/post-filed/网页内容接收回调函数等  
3. curl_easy_perform执行  
4. curl_easy_cleanup清理  
注意在第3步是阻塞的

##### 1.2 Multi Interface

Multi下是异步接口，curl_multi_*的形式，允许在单线程下同时操作多个easy handle，基本处理方式有几个步骤：  
1. curl_multi_init获取multi handle  
2. 调用Easy Interface设置若干easy handle，通过curl_multi_add_handle.加到1里的multi handle  
3. 调用curl_multi_wait/curl_multi_perform等待所有easy handle处理完成  
4. curl_multi_info_read依次读取所有数据  
5. curl_multi/easy_cleanup清理  
注意curl_multi_perform不是阻塞的

##### 1.3 Share Interface
Share是共享接口，curl_shared_*的形式，用于多个easy handle间共享一些数据，例如cookie dns等  
注意需要用到锁的情况，比如share了CURL_LOCK_DATA_DNS，如果没有加锁，在`curl_multi_perform`时会core掉:

> Since you can use this share from multiple threads, and libcurl has no internal thread synchronization, you must provide mutex callbacks if you're using this multi-threaded. You set lock and unlock functions with curl_share_setopt too.

可以参考这两处例子：  
1. <http://www.mit.edu/afs.new/sipb/user/ssen/src/curl-7.11.1/tests/libtest/lib506.c>  
2. <https://curl.haxx.se/mail/lib-2006-01/0218.html>  

#### 2. 异步接口的例子

curl自带的[例子](https://curl.haxx.se/libcurl/c/multi-app.html)还是介绍的curl_multi_fdset的方法。  
实际上已经可以用curl_multi_wait代替了，据说是facebook的工程师提的升级：  
[Facebook contributes fix to libcurl’s multi interface to overcome problem with more than 1024 file descriptors.](https://daniel.haxx.se/blog/2012/09/03/introducing-curl_multi_wait/)  
使用方法可以参考[这里](https://gist.github.com/clemensg/4960504)

通过一个示例来看下multi的工作方式（注意出于简短的目的，有的函数没有判断返回值）

```cpp
#include "curl/curl.h"
#include <string>

const char* url = "http://www.baidu.com";
const int count = 1000;

size_t write_data(void* buffer, size_t size, size_t count, void* stream) {
    (void)buffer;
    (void)stream;
    return size * count;
}

void curl_multi_demo() {
    CURLM* curlm = curl_multi_init();

    for (int i = 0; i < count; ++i) {
        CURL* easy_handle = curl_easy_init();
        curl_easy_setopt(easy_handle, CURLOPT_NOSIGNAL, 1);
        curl_easy_setopt(easy_handle, CURLOPT_URL, url);
        curl_easy_setopt(easy_handle, CURLOPT_WRITEFUNCTION, write_data);

        curl_multi_add_handle(curlm, easy_handle);
    }

    int running_handlers = 0;
    do {
        curl_multi_wait(curlm, NULL, 0, 2000, NULL);
        curl_multi_perform(curlm, &running_handlers);
    } while (running_handlers > 0);

    int msgs_left = 0;
    CURLMsg* msg = NULL;
    while ((msg = curl_multi_info_read(curlm, &msgs_left)) != NULL) {
        if (msg->msg == CURLMSG_DONE) {
            int http_status_code = 0;
            curl_easy_getinfo(msg->easy_handle, CURLINFO_RESPONSE_CODE, &http_status_code);
            char* effective_url = NULL;
            curl_easy_getinfo(msg->easy_handle, CURLINFO_EFFECTIVE_URL, &effective_url);
            fprintf(stdout, "url:%s status:%d %s\n",
                    effective_url,
                    http_status_code,
                    curl_easy_strerror(msg->data.result));

            curl_multi_remove_handle(curlm, msg->easy_handle);
            curl_easy_cleanup(msg->easy_handle);
        }
    }

    curl_multi_cleanup(curlm);
}

int main() {
    curl_multi_demo();
    return 0;
}
```
**注意L35~36 L37~38不能互换，否则url为空，原因没有继续深追。**


可以看到异步处理的方式是通过`curl_multi_add_handle`接口不断的把待抓的easy handle放到multi handle里。然后通过`curl_multi_wait/curl_multi_perform`等待所有easy handle处理完毕。  
因为是同时在等待所有easy handle处理完毕，耗时比easy方式里逐个同步等待大大减少。其中产生hold作用的是在这段代码里：

```cpp
//等待所有easy handle处理完毕
do {
    ...
} while (running_handlers > 0);
```

#### 3.应用

而在实际应用中，我们的使用场景往往是这样的：  

**某个负责抓取数据的模块，service监听端口接收url，抓取数据后发往下游。**

因为不希望所有的线程都处于上面multi示例中的等待（什么都不做）。于是就有了这种想法：接收到一条url后构造对应的easy handle，通过curl_multi_add_handle接口放入curlm后返回。同时两个线程不断的wait/perform和read，如果有完成的url，那么就调用对应的回调函数即可。  
相当于将curlm当做一个队列，两个线程分别充当了生产者和消费者。  

模型类似于：

```cpp
CURLM* curlm = curl_multi_init()

//Thread-Consumer
	while true:
		//读取已经完成的url
		curl_multi_info_read
		//通知该url已完成，并且从curlm里删除
		curl_multi_remove_handle
		
//Thread-Producer
	while true:
		//等待可读socket
		curl_multi_wait
		//查看运行中的easy handle数目
		curl_multi_perform
		
//Thread-Other
	//根据url构造CURL easy handle
	CURL* curl = make_curl_easy_handle(url)
	//添加到curlm
	curl_multi_add_handle
```

做了一下测试很快就放弃了，程序在libcurl内部函数里core掉。  
实际上curl是线程安全的，但同时也格外强调了这点：  

> The first basic rule is that you must never share a libcurl handle (be it easy or multi or whatever) between multiple threads. Only use one handle in one thread at a time.

具体可以参考[这里](https://curl.haxx.se/libcurl/c/threadsafe.html)，说明上面的模型是不可行的。

看到这里有一个[基于libcurl的单线程I:O多路复用HTTP框架](http://iosqqmail.github.io/2016/03/01/%E5%9F%BA%E4%BA%8Elibcurl%E7%9A%84%E5%8D%95%E7%BA%BF%E7%A8%8BI:O%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8HTTP%E6%A1%86%E6%9E%B6/)，dispatch部分和chrome源码里的thread模型很像，CURLM*对象在Dispatch::IO线程里统一操作，同时全局唯一，在一个LazyInstance的ConnectionRunner里维护。不过没有找到`dispatch_after`的实现，所以不太确定。   

在StackOverflow上看到了[复用curl的想法](http://stackoverflow.com/questions/15870395/using-libcurl-from-multiple-threads-how-to-get-the-best-performance)：curl handler放在一个池子中，需要时从中获取，使用后归还，同样不可行。

因此，标准的写法就是之前的示例的代码，正如[这里](http://stackoverflow.com/questions/6900222/multithreaded-libcurl)提到的：  

> The multi interface is designed for this purpose: you add multiple handles and then process all of them with one call, all in the same thread.

#### 4.优化
接下来就是优化的问题，在不使用curl_multi_socket_*的接口的情况下，是否有办法提升性能呢？  
参考了curl的[Persistence](https://curl.haxx.se/libcurl/c/libcurl-tutorial.html#Persistence
)一节，主要是持久化部分信息来加速（缓存）。  
其中提到  

> Each easy handle will attempt to keep the last few connections alive for a while in case they are to be used again.

这里说到每个easy handle会缓存之前的若干连接来避免重连、缓存DNS等以提高性能。因此一些思路就是easy handle重用、dns全局缓存等。  

#### 5.测试&结论

按照上面的思路分别测试抓取1000次baidu首页  
1.	串行使用curl easy接口  
2.	10个线程并行使用curl easy接口  
3.	使用curl multi接口  
4.	使用curl multi接口,并且reuse connection。（方法是第一遍curl easy handle抓取后不cleanup，计算第二次全部抓取完成的时间）  
5.	使用dns cache（使用curl_share_*接口，第一次抓取用于dnscache填充，计算第二次全部抓取完成的时间。效果上打开VERBOSE可以看到hostname found in DNS Cache）  

其中处理时间测试结论如下：

|method            |avg      |max      |min      |
|------------------|---------|---------|---------|
|easy              |16.457   |21.617   |14.262   |
|easy parallel     |2.331    |9.496    |1.723    |
|multi             |0.734    |8.895    |0.259    |
|multi reuse       |0.00113  |0.001557 |0.000898 |
|multi reuse cache |0.00109  |0.00140  |0.000828 |

4 5的区别不大，同时不确定重用connection的情况下，dnscache是否还能起到正向作用

对应的测试代码都放到了gist上：[1](https://gist.github.com/yingshin/cd1df34c84e832cfc1fc314d8145c259) [2](https://gist.github.com/yingshin/c33725da0b85f0e98ade38ed3a684b9b) [3](https://gist.github.com/yingshin/ce0331c9c1f842ddb636341597c99a7a) [4](https://gist.github.com/yingshin/0f7d84799f7743ca4757c4b5edf6c1bc) [5](https://gist.github.com/yingshin/ad8cb57588cd9418a6d6d20af7135ddd)


-------------------------------

#### 6.补充

关于curl性能的讨论帖子很多，比如[这里](http://www.linuxquestions.org/questions/programming-9/looking-for-open-source-library-better-than-libcurl-767195/)，其中也讲到了获取网页的一个基本流程：

1. Request from your DNS server, the IP corresponding to the name of the site you requested
2. Use the server's reply to open a socket to that IP, port 80
3. Send a small HTTP message describing what you want
4. Receive the html code

[这里](https://moz.com/devblog/high-performance-libcurl-tips/)有一些关于performance的建议，用到了curl_multi_socket_*接口，我没有用到。
