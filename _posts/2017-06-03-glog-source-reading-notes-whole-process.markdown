---
title: glog源码解析二：从LOG(INFO)到写入日志的代码分析
date: 2017-06-03 22:50:14
excerpt: "glog源码解析"
tags: [glog]
---

这篇笔记接着[上篇笔记:glog源码解析一：整体结构](http://izualzhy.cn/glog-source-reading-notes)，继续介绍`LOG(INFO)`调用后的完整过程，通过梳理这个过程，能覆盖到大部分的代码。当然为了便于理解，一些细节和分支都做了简化。

<!--more-->

上篇文章讲到了宏替换的过程，比如

```cpp
LOG(INFO) << "hello world.";
```

会被替换成

```cpp
LogMessage(__FILE__, __LINE__).stream() << "hello world.";
```
虽然只是简单的一行代码，却完成了日志数据的写入，这其中实际上经过了复杂的函数调用关系。

为了更清晰的说明源码的结构，我把整个调用过程分成了三部分：日志数据的接收、分发、存储。

## 1. 日志数据的接收

`LogMessage`是日志库对外的接口类，存在多个构造函数

```cpp
  LogMessage(const char* file, int line, LogSeverity severity, int ctr,
             SendMethod send_method);
  LogMessage(const char* file, int line);
  LogMessage(const char* file, int line, LogSeverity severity);
  LogMessage(const char* file, int line, LogSeverity severity, LogSink* sink,
             bool also_send_to_log);
  LogMessage(const char* file, int line, LogSeverity severity,
             std::vector<std::string>* outvec);
  LogMessage(const char* file, int line, LogSeverity severity,
             std::string* message);
  LogMessage(const char* file, int line, const CheckOpString& result);
```

这么多构造函数实际上是为了众多的宏准备的，这些宏定义使得面向用户的接口十分灵活。有些宏会在接下来其他笔记里提到，这里`LOG(INFO) LOG(WARNING)`替换后对应的构造函数最为简单，我们也从这两个最标准的构造函数入手。

```cpp
LogMessage::LogMessage(const char* file, int line)
    : allocated_(NULL) {
  Init(file, line, GLOG_INFO, &LogMessage::SendToLog);
}

LogMessage::LogMessage(const char* file, int line, LogSeverity severity)
    : allocated_(NULL) {
  Init(file, line, severity, &LogMessage::SendToLog);
}
```

注意`LogMessage::SendToLog`这个函数，`Init`时会传入，根据构造函数的不同选择不同的成员函数。我们在下一节会重点介绍下。

实际上所有的构造函数都会调用到`Init`这个接口:

```cpp
void LogMessage::Init(const char* file,
                      int line,
                      LogSeverity severity,
                      void (LogMessage::*send_method)()) {
    //堆上分配空间并初始化LogMessageData* data_;
    //初始化内容包括severity, ts, line, filename等
    //日志数据增加logprefix，即severity，ts，线程id(syscall(__NR_gettid))，文件basename
}
//注：忽略了fatal日志特殊处理，保留errno，判断是否需要logprefix等代码，只介绍主要流程部分。本文其他地方也是如此。
```

`LogMessageData`，是一个负责存储日志数据以及组织写入的类。

部分成员变量负责存储日志数据:

```cpp
  // Buffer space; contains complete message text.
  char message_text_[LogMessage::kMaxLogMessageLen+1];
  LogStream stream_;
  char severity_;      // What level is this LogMessage logged at?
  int line_;                 // line number where logging call is.
```

其中`LogStream stream_`负责流式数据的写入，继承自`std::ostream`，同时实现了自己的`streambuf`。数据缓冲区使用`message_text_`，用于接收流式的日志数据并存储。可以看到`message_text_`是定长的字符串。关于这个长度注释也比较有意思（大意就是这个魔数你就别猜原因了，就是个武断的值）：

```cpp
// An arbitrary limit on the length of a single log message.  This
// is so that streaming can be done more efficiently.
const size_t LogMessage::kMaxLogMessageLen = 30000;
```

因此glog支持的单行日志不能超过30000个字节。

而负责组织写入主要有两个成员变量：

```cpp
  void (LogMessage::*send_method_)();  // Call this in destructor to send
  union {  // At most one of these is used: union to keep the size low.
    LogSink* sink_;             // NULL or sink to send message to
    std::vector<std::string>* outvec_; // NULL or vector to push message onto
    std::string* message_;             // NULL or string to write message into
  };
```

`send_method_`在`LogMessage::Init`时赋值，可能的函数有：

```cpp
LogMessage::SendToLog
LogMessage::SendToSinkAndLog
LogMessage::SendToSinkToLog
LogMessage::WriteToStringAndLog
```

`sink_ outvec_ messag_`等也是通过各种LOG宏，构造`LogMessage`时通过`Init`传入，用于用户自定义的输出方式。

日志数据的后续接收通过`LogStream stream_;`完成，由于继承自`std::ostream`因此天然支持`<<`的流式日志输入，"hello world."字符串最后存储到了`message_text_`。

## 2. 日志数据的分发

数据存储到`message_text_`之后，在`~LogMessage`的析构函数里进行分发与落盘。

```cpp
LogMessage::~LogMessage() {
  Flush();
  delete allocated_;
}
```

`Flush`的实现：

```cpp
void LogMessage::Flush() {
    //根据 FLAGS_minloglevel 判断是否需要分发，如果不需要，则直接返回
    //是否需要添加\n（感兴趣的读者可以试下"hello\n" "hello"实际上输出了相同的日志 ）
    // Prevent any subtle race conditions by wrapping a mutex lock around
    // the actual logging action per se.
    {
        MutexLock l(&log_mutex);
        (this->*(data_->send_method_))();//调用send_method_分发数据到不同的日志目的地
        ++num_messages_[static_cast<int>(data_->severity_)];//为每种日志级别增加计数
    }
    LogDestination::WaitForSinks(data_);//等待sink完成
```

注意这里才对`FLAGS_minloglevel`的判断，因此设置了日志级别`FLAGS_minloglevel`只能保证不再输出，`LogMessage`仍然会构造出来，并且执行后面的代码。

例如这段代码

```cpp
FLAGS_minlogleve = 1;
int a = 1;
LOG(INFO) << a++;
```

虽然INFO日志不会输出，但是这条语句使得`a`的值发生了变化:1 -> 2。

前面介绍过`send_method_`可能有各种赋值，不过都来源于`LogMessage`的成员函数，对于普通的日志输出，`send_method_ = &LogMessage::SendToLog`，我们看下`LogMessage::SendToLog`的定义

```cpp
  ...
  // global flag: never log to file if set.  Also -- don't log to a
  // file if we haven't parsed the command line flags to get the
  // program name.
  if (FLAGS_logtostderr || !IsGoogleLoggingInitialized()) {
    ColoredWriteToStderr(data_->severity_,
                         data_->message_text_, data_->num_chars_to_log_);

    // this could be protected by a flag if necessary.
    LogDestination::LogToSinks(data_->severity_,
                               data_->fullname_, data_->basename_,
                               data_->line_, &data_->tm_time_,
                               data_->message_text_ + data_->num_prefix_chars_,
                               (data_->num_chars_to_log_ -
                                data_->num_prefix_chars_ - 1));
  } else {

    // log this message to all log files of severity <= severity_
    // 写日志到各级别的日志文件
    LogDestination::LogToAllLogfiles(data_->severity_, data_->timestamp_,
                                     data_->message_text_,
                                     data_->num_chars_to_log_);

    LogDestination::MaybeLogToStderr(data_->severity_, data_->message_text_,
                                     data_->num_chars_to_log_);
    LogDestination::MaybeLogToEmail(data_->severity_, data_->message_text_,
                                    data_->num_chars_to_log_);
    //写到用户自定义的sink输出，message_text_去掉了默认添加的logprefix，即severity，ts，线程id，文件basename等
    LogDestination::LogToSinks(data_->severity_,
                               data_->fullname_, data_->basename_,
                               data_->line_, &data_->tm_time_,
                               data_->message_text_ + data_->num_prefix_chars_,
                               (data_->num_chars_to_log_
                                - data_->num_prefix_chars_ - 1));
    // NOTE: -1 removes trailing \n
  }
  ...
```

上面只摘抄了中间最重要的部分代码，可以看到这里通过判断是否已经初始化等进行日志目的地的分发，日志被分发到`LogDestination`的各个静态函数。

介绍下`LogToAllLogfiles`和`LogToSinks`：

```cpp
inline void LogDestination::LogToAllLogfiles(LogSeverity severity,
                                             time_t timestamp,
                                             const char* message,
                                             size_t len) {

  if ( FLAGS_logtostderr ) {           // global flag: never log to file
    ColoredWriteToStderr(severity, message, len);
  } else {
    //从[severity, 0]全部输出，因此我们可以看到warning日志会出现在info日志里。
    for (int i = severity; i >= 0; --i)
      LogDestination::MaybeLogToLogfile(i, timestamp, message, len);
  }
}
```

在`logging.cc`里定义了一个全局的数组`LogDestination* LogDestination::log_destinations_[NUM_SEVERITIES];`，每种日志级别对应其中一个元素，通过静态函数`LogDestination::log_destination`访问。

```cpp
inline LogDestination* LogDestination::log_destination(LogSeverity severity) {
  assert(severity >=0 && severity < NUM_SEVERITIES);
  if (!log_destinations_[severity]) {
    log_destinations_[severity] = new LogDestination(severity, NULL);
  }
  return log_destinations_[severity];
}
```

`log_destinations_`可以灵活的设置，这里先继续介绍上面标准流程里的`LogToSinks`。

```cpp
inline void LogDestination::LogToSinks(LogSeverity severity,
                                       const char *full_filename,
                                       const char *base_filename,
                                       int line,
                                       const struct ::tm* tm_time,
                                       const char* message,
                                       size_t message_len) {
  ReaderMutexLock l(&sink_mutex_);
  if (sinks_) {
      //依次调用所有sinks_里元素的send方法
    for (int i = sinks_->size() - 1; i >= 0; i--) {
      (*sinks_)[i]->send(severity, full_filename, base_filename,
                         line, tm_time, message, message_len);
    }
  }
}
```

`sinks_`是`LogDestination`的静态成员变量，用于存储用户自定义的输出方式。

```cpp
static vector<LogSink*>* sinks
```

因此如果要自定义输出方式，我们只要继承`LogSink`，然后override `LogSink::send`方法即可。

## 3. 日志数据的存储

接着上节`LogDestination::MaybeLogToLogfile`的调用讲起

```cpp
inline void LogDestination::MaybeLogToLogfile(LogSeverity severity,
        time_t timestamp,
        const char* message,
        size_t len) {
    //判断是立即flush还是先缓存，logbuflevel默认值=0，
    //各日志级别的定义：const int GLOG_INFO = 0, GLOG_WARNING = 1, GLOG_ERROR = 2, GLOG_FATAL = 3
    //可以看到默认只会对INFO级别缓存,should_flush = false
    const bool should_flush = severity > FLAGS_logbuflevel;
    //从log_destinations_数组获取到该级别对应的LogDestination*
    LogDestination* destination = log_destination(severity);
    //完成日志的写入
    destination->logger_->Write(should_flush, timestamp, message, len);
}
```

`logger_->Write`完成了日志写入，`LogDestination`实际上只有两个成员变量：

```cpp
  LogFileObject fileobject_;
  base::Logger* logger_;      // Either &fileobject_, or wrapper around it
```

其中`logger_ = &fileobject_`, `fileobejct_`实现了默认的日志输出方式。

这种设计方式使得我们可以重新绑定`logger_`，指向其他`LogFileObject`或者自定义的`base::Logger`子类, 通过`base::SetLogger`接口可以实现这点，也就改变了标准日志输出的方式。

```cpp
void base::SetLogger(LogSeverity severity, base::Logger* logger) {
  MutexLock l(&log_mutex);
  LogDestination::log_destination(severity)->logger_ = logger;
}
```

`LogFileObject::write`函数负责创建日志文件、写入等：

```cpp
  //加锁，避免同时操作同一文件
  MutexLock l(&lock_);
  //是否需要重新创建日志文件
  if (static_cast<int>(file_length_ >> 20) >= MaxLogSize() ||
      PidHasChanged()) {
  //调用fwrite写入日志
  //判断是否需要fflush落盘
  // See important msgs *now*.  Also, flush logs at least every 10^6 chars,
  // or every "FLAGS_logbufsecs" seconds.
  if ( force_flush ||
       (bytes_since_flush_ >= 1000000) ||
       (CycleClock_Now() >= next_flush_time_) ) {
    FlushUnlocked();
  ...
```

标准日志的默认输出方式到这里就结束了。对于用户自定义的输出方式，则是通过`LogDestination::LogToSinks`完成。

用户自定义的`LogSink`通过`static vector<LogSink>* sinks_`管理，添加和删除对应的`LogSink*`的接口如下：

```cpp
inline void LogDestination::AddLogSink(LogSink *destination);
inline void LogDestination::RemoveLogSink(LogSink *destination);
```

基于这两个接口，再通过继承`LogSink`实现对应的子类就可以添加自定义日志输出方式了。

下一节我们基于上面的分析来实现glog功能的修改与扩展。
