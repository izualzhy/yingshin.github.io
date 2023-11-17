---
title: "leveldb笔记之22:Better Code"
date: 2019-04-05 10:27:05
tags: leveldb
---

本文总结下 leveldb 一些好的代码习惯。

## 1. 返回信息丰富

`leveldb::Status`除了返回码，可以提供更丰富的返回信息。

例如,`leveldb::DB::Put`接口的返回值：

```
  virtual Status Put(const WriteOptions& options,
                     const Slice& key,
                     const Slice& value) = 0;
```

内部使用`const char*`来记录全部内容：

```
  // OK status has a null state_.  Otherwise, state_ is a new[] array
  // of the following form:
  //    state_[0..3] == length of message
  //    state_[4]    == code
  //    state_[5..]  == message
  const char* state_;

  enum Code {
    kOk = 0,
    kNotFound = 1,
    kCorruption = 2,
    kNotSupported = 3,
    kInvalidArgument = 4,
    kIOError = 5
  };
```

在 brpc 里也有类似的实现：[butil::Status](https://github.com/apache/incubator-brpc/blob/master/src/butil/status.h)

## 2. 抽象接口

典型的就是[Iterator](https://izualzhy.cn/leveldb-iterator)，算法作用于容器，但通过接口的抽象，算法的实现不必依赖容器。

## 3. 单一职责

最近跟一位大牛聊天时，提到了 posix 的文件接口设计，简洁而且高效。让我想起来某次听到的 Jeff Dean 对 Sanjay 的接口设计的评价：

当你需要某个接口时，发现他就在那里，不多不少。

我写过也 review 过很多文件操作的代码，很多时候都会有`File` `FileHandle`的类，杂糅了诸如文件读&写、文件/目录创建等功能。leveldb 里的设计是一个比价好的典范：

```
// A file abstraction for reading sequentially through a file
class LEVELDB_EXPORT SequentialFile {
    virtual Status Read(size_t n, Slice* result, char* scratch) = 0;
    virtual Status Skip(uint64_t n) = 0;
};

// A file abstraction for randomly reading the contents of a file.
class LEVELDB_EXPORT RandomAccessFile {
  virtual Status Read(uint64_t offset, size_t n, Slice* result,
                      char* scratch) const = 0;
};

// A file abstraction for sequential writing.  The implementation
// must provide buffering since callers may append small fragments
// at a time to the file.
class LEVELDB_EXPORT WritableFile {
  virtual Status Append(const Slice& data) = 0;
  virtual Status Close() = 0;
  virtual Status Flush() = 0;
  virtual Status Sync() = 0;
}
```

良好的类设计不仅仅是对于《设计模式》的熟悉，很多时候是对跨平台的熟悉、模块功能的理解。例如文件操作接口这么设计，而打开文件`fd`则是平台的操作：

```
class PosixEnv : public Env {
  ...
  //只读方式打开文件fname，通过SequentialFile读取文件内容，*result指向该对象
  //对象析构时close文件句柄
  virtual Status NewSequentialFile(const std::string& fname,
                                   SequentialFile** result) {
    int fd = open(fname.c_str(), O_RDONLY);
    if (fd < 0) {
      *result = nullptr;
      return PosixError(fname, errno);
    } else {
      *result = new PosixSequentialFile(fname, fd);
      return Status::OK();
    }
  }
```

## 4. call_once

当提供一个 lib 服务时，依赖方式会变得复杂，例如可能这样：

```
libA.a depends on libhelloworld.a
libB.a depends on libhelloworld.a
executable file C depends on libA.a && libB
```

由于 libA libB 都用到了 libhelloworld.a，lib 本身就需要考虑多次初始化，同时初始化的问题。

`std::call_once` `pthread_once`都是不错的选择，场景比如[new 全局唯一的 BytewiseComparator](https://github.com/yingshin/leveldb_more_annotation/blob/master/util/comparator.cc)

真实的 C++ 项目暴露出的问题会更隐蔽、更复杂，例如之前关于[全局变量core的问题](https://izualzhy.cn/double-free-with-global-variable)，就是在 pb3 里采用了`GoogleOnceInit`来解决。不过刚才看时，发现又换了一种方式：[Fix initialization with Visual Studio #4878
](https://github.com/protocolbuffers/protobuf/pull/4878/commits/a9abc7831e45257d334cfa682746b6cadf9e95d9)

当然，什么时候应当考虑这个问题，又是工程实践上的另一个话题：什么是过度设计 and 什么是前瞻性的设计？

## 5. 引用计数

例如

```
void Version::Ref() {
  ++refs_;
}

void Version::Unref() {
  assert(this != &vset_->dummy_versions_);
  assert(refs_ >= 1);
  --refs_;
  if (refs_ == 0) {
    delete this;
  }
}
```

其实就是`shared_ptr`的思路，当然也有更复杂一点的：

```
void LRUCache::Unref(LRUHandle* e) {
  assert(e->refs > 0);
  e->refs--;
  // 外部也没有持有e，可以删除
  if (e->refs == 0) {  // Deallocate.
    assert(!e->in_cache);
    (*e->deleter)(e->key(), e->value);
    free(e);
  } else if (e->in_cache && e->refs == 1) {
    // in_cache为true且引用计数为1，说明只有in_cache还持有e
    // 表示节点可以由对象本身控制，因此移动到lru_
    // No longer in use; move to lru_ list.
    LRU_Remove(e);
    LRU_Append(&lru_, e);
  }
}
```

不知道是不是因为这个原因，没有使用`shared_ptr`。

不过无论如何，假设去掉`refs_`，资源的析构时间在多线程各种条件下，一定是段复杂难以阅读的代码

## 6. 注释

注释的唯一目标是为了可读。

注释非常多并不一定更可读，反而读起来更费劲。过期不一致的注释，更带来混淆的副作用。注释没有统一的标准，有人认为注释是对代码的补充，有人认为所谓的自注释代码是扯淡，因为注释的是模块的思路。

所以注释的目标尽管明确，但是怎么做却不容易说清楚。

与传统软件开发的流程不同，互联网的研发模式的核心是“快”，所以可以锻炼 rd 快速迭代、创新、实现某个功能的能力，相反的，并不会提高写注释的能力。所以会发现一个现象，多年的 rd 写的注释，甚至可能没有应届生写的好(仔细)，可能大厂确实好一点，但是也不要抱太大希望。

扯这么多蛋，只是为了说清楚：这个话题很重要，但我说不清楚o(╯□╰)o

不过举个例子，`if else`的代码是避免不了的，看下 leveldb 是如何注释的：

```
  while (true) {
    if (!bg_error_.ok()) {
      // Yield previous error
      s = bg_error_;
      break;
    } else if (
        allow_delay &&
        versions_->NumLevelFiles(0) >= config::kL0_SlowdownWritesTrigger) {
      // We are getting close to hitting a hard limit on the number of
      // L0 files.  Rather than delaying a single write by several
      // seconds when we hit the hard limit, start delaying each
      // individual write by 1ms to reduce latency variance.  Also,
      // this delay hands over some CPU to the compaction thread in
      // case it is sharing the same core as the writer.
      // 0层文件超过8个，则等待，至多等待一次
      mutex_.Unlock();
      env_->SleepForMicroseconds(1000);
      allow_delay = false;  // Do not delay a single write more than once
      mutex_.Lock();
    } else if (!force &&
               (mem_->ApproximateMemoryUsage() <= options_.write_buffer_size)) {
      // mem_不足4M，可以继续写入
      // There is room in current memtable
      break;
    } else if (imm_ != nullptr) {
      // We have filled up the current memtable, but the previous
      // one is still being compacted, so we wait.
      Log(options_.info_log, "Current memtable full; waiting...\n");
      background_work_finished_signal_.Wait();
    } else if (versions_->NumLevelFiles(0) >= config::kL0_StopWritesTrigger) {
      // There are too many level-0 files.
      // level-0文件个数需要控制，避免影响查找速度
      // 因此>=12个，则停止写入
      Log(options_.info_log, "Too many L0 files; waiting...\n");
      background_work_finished_signal_.Wait();
    } else {
      // Attempt to switch to a new memtable and trigger compaction of old
      ...
```

## 7. mutable

如果类的成员函数没有修改内部变量，标记为 const 函数，对外/内使用时都会更放心一些。

```
// A single shard of sharded cache.
class LRUCache {
 ...
  size_t TotalCharge() const {
    MutexLock l(&mutex_);
    return usage_;
  }
```

而这里常见的问题是 const 函数内可能有加锁操作，加锁本身是非 const 的行为，因此可以设置为 mutable 以支持编译通过：

```
mutable port::Mutex mutex_;
```

## 单测

留个坑，慢慢补充
