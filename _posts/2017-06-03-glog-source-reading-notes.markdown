---
title: glog源码解析一：整体结构
date: 2017-06-03 20:10:11
excerpt: "glog源码解析"
tags: [glog]
---

上次写完一篇[glog使用的笔记](http://izualzhy.cn/glog)后，一直想抽时间分析下[glog](https://github.com/google/glog)的源码。因为实际上glog提供了比笔记里多得多的功能与接口，通过阅读源码可以更深入的理解如何使用这些接口。

从这篇笔记开始，会逐渐开始介绍glog源码阅读的一些心得。

本文首先从`LOG(INFO)`入手，看下相关调用过程，之后介绍下源码里各个类的作用。

<!--more-->

## 1. LOG(INFO)的宏替换

glog相关源码都在src下面，并不存在我们会经常include的`logging.h`文件。但是可以找到`logging.h.in`，这个文件通过`configure`的替换，生成`logging.h`，替换内容包括一些#include，是否支持gflags等。

在`logging.h`，可以看到`LOG(xxx)`实际上通过宏定义转换为了类`LogMessage`的操作。

```
#define LOG(severity) COMPACT_GOOGLE_LOG_ ## severity.stream()
```

`##`起到连接符的作用，`COMPACT_GOOGLE_LOG_XXX.stream()`也是一个宏定义。

```
#define COMPACT_GOOGLE_LOG_INFO google::LogMessage( \
    __FILE__, __LINE__)
#define COMPACT_GOOGLE_LOG_WARNING google::LogMessage( \
    __FILE__, __LINE__, google::GLOG_WARNING)
```

可以看到针对不同的日志级别(severity)，实际上是调用了`LogMessage`不同的构造函数。（注意源码里要复杂一些，比如会判断`GOOGLE_STRIP_LOG`，这里为了方便说明做了简化）

到这一步，都是通过宏定义完成的。

虽然我们在现代C++代码里应当尽量避免宏的使用，但是实际上我们总是离不开宏定义，比如这里因为需要获取到`LOG(xxx)`调用处的文件名(__FILE__)，行号(__LINE__)，就必须使用宏的替换。

到这里，我们可以看到`LOG(xxx)`最后替换完成的样子

```
LOG(INFO) => COMPACT_GOOGLE_LOG_INFO.stream() => LogMessage(__FILE__, __LINE__).stream()
LOG(WARNING) => COMPACT_GOOGLE_LOG_WARNING.stream() => LogMessage(__FILE__, __LINE__, google::GLOG_WARNING).stream()
```

`stream`是`LogMessage`的public函数

```
ostream& LogMessage::stream() {
  return data_->stream_;
}
```

那么`data_ data_->stream_`是什么？我们看下glog里重要的几个类图：

## 2. glog_uml 类图
![glog_uml](/assets/images/glog_uml.png)

1. `LogMessage`:日志库的接口部分，在前面已经见到过了。提供了多个构造函数，在析构时调用`Flush`写入日志数据。也就是每次`LOG(xxx)`的调用都会生成一个`LogMessage`对象。同时对象记录了写入日志数据的函数指针：`send_method`，其中数据的存储和写入日志都委托给`LogMessageData`完成。
2. `LogMessageData`：记录日志数据例如文件名、日志消息、日志级别、行号、时间等,同时调用`LogDestination`的静态方法组织数据的写入。
3. `LogStream`继承自`std::ostream`，glog使用时`<<`流式输入日志的奥秘就来源于`LogStream`。
4. `LogStreamBuf`继承自`std::streambuf`，`LogStream`使用`rdbuf`接口设置使用该streambuffer，其中缓冲区buffer为`message_text_`，由`LogMessageData`管理。关于`std::streambuf`可以参考[这篇笔记](http://izualzhy.cn/stream-buffer)。
5. `LogDestination`管理了输出对象，包括两类：默认输出和用户自定义输出，其中默认输出方式的对象为`LogDestination`本身，对每种日志级别都有一个全局的`LogDestination`对象负责日志的写入。用户自定义输出对象为`LogSink`。
6. `LogFileObject` 通过`write`方法完成数据真正写入到具体文件，也就是将INFO WARNING等日志真正写入文件的类。
7. `LogSink` 虚基类，用户可以继承该类并且override `send`方法，就可以实现自定义的日志输出方式了。
8. `Logger` `LogFileObject`的基类，通过继承该类可以修改默认的日志输出方式。
