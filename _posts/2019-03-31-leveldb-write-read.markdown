---
title: "leveldb笔记之20:写入与读取流程"
date: 2019-03-31 12:23:09
tags: leveldb
---

这篇笔记介绍下写入和读取过程，前面已经铺垫了很多基础组件，写入介绍起来相对简单一些了。

## 1. Put

先用一张图片介绍下：

![Write](assets/images/leveldb/Write.png)

写入的`key value`首先被封装到`WriteBatch`

```cpp
// Default implementations of convenience methods that subclasses of DB
// can call if they wish
Status DB::Put(const WriteOptions& opt, const Slice& key, const Slice& value) {
  WriteBatch batch;
  //key,value数据更新到batch里
  batch.Put(key, value);
  return Write(opt, &batch);
}
```

`WriterBatch`封装了数据，`DBImpl::Writer`则继续封装了 mutex cond 等同步原语

```cpp
// Information kept for every waiting writer
struct DBImpl::Writer {
  Status status;
  WriteBatch* batch;
  bool sync;
  bool done;
  port::CondVar cv;

  explicit Writer(port::Mutex* mu) : cv(mu) { }
};
```

写入流程实际上调用的是`DBImpl::Write`

```cpp
//调用流程: DBImpl::Put -> DB::Put -> DBImpl::Write
Status DBImpl::Write(const WriteOptions& options, WriteBatch* my_batch) {
  //一次Write写入内容会首先封装到Writer里，Writer同时记录是否完成写入、触发Writer写入的条件变量等
  Writer w(&mutex_);
  w.batch = my_batch;
  w.sync = options.sync;
  w.done = false;
```

数据被写入到`writers_`，直到满足两个条件：  
1. 其他线程已经帮忙完成了`w`的写入  
2. 抢到锁并且位于`writers_`首部  

```cpp
  MutexLock l(&mutex_);//多个线程调用的写入操作通过mutex_串行化
  writers_.push_back(&w);
  //数据先放到queue里，如果不在queue顶部则等待
  //这里是对数据流的一个优化，wirters_里Writer写入时，可能会把queue里其他Writer也完成写入
  while (!w.done && &w != writers_.front()) {
    w.cv.Wait();
  }
  //如果醒来并且抢到了mutex_，检查是否已经完成了写入(by其他Writer)，则直接返回写入status
  if (w.done) {
    return w.status;
  }
```

接着查看是否有足够空间写入，例如`mem_`是否写满，是否必须触发 minor compaction 等

```cpp
  // May temporarily unlock and wait.
  Status status = MakeRoomForWrite(my_batch == nullptr);
```

取出`writers_`的数据，统一记录到`updates`

```cpp
  uint64_t last_sequence = versions_->LastSequence();//本次写入的SequenceNumber
  Writer* last_writer = &w;
  if (status.ok() && my_batch != nullptr) {  // nullptr batch is for compactions
    //updates存储合并后的所有WriteBatch
    WriteBatch* updates = BuildBatchGroup(&last_writer);
    WriteBatchInternal::SetSequence(updates, last_sequence + 1);
    last_sequence += WriteBatchInternal::Count(updates);
```

然后[写入日志](https://izualzhy.cn/leveldb-log)，[写入内存](https://izualzhy.cn/memtable-leveldb):

```cpp
      //WriterBatch写入log文件，包括:sequence,操作count,每次操作的类型(Put/Delete)，key/value及其长度
      status = log_->AddRecord(WriteBatchInternal::Contents(updates));
      bool sync_error = false;
      if (status.ok() && options.sync) {
        //log_底层使用logfile_与文件系统交互，调用Sync完成写入
        status = logfile_->Sync();
        if (!status.ok()) {
          sync_error = true;
        }
      }
      //写入文件系统后不用担心数据丢失，继续插入MemTable
      if (status.ok()) {
        status = WriteBatchInternal::InsertInto(updates, mem_);
      }
```

写入完成后，逐个唤醒等待的线程:

```cpp
  //last_writer记录了writers_里合并的最后一个Writer
  //逐个遍历弹出writers_里的元素，并环形等待write的线程，直到遇到last_writer
  while (true) {
    Writer* ready = writers_.front();
    writers_.pop_front();
    if (ready != &w) {
      ready->status = status;
      ready->done = true;
      ready->cv.Signal();
    }
    if (ready == last_writer) break;
  }

  // Notify new head of write queue
  // 唤醒队列未写入的第一个Writer
  if (!writers_.empty()) {
    writers_.front()->cv.Signal();
  }
```

## 2. Sequence

批量写入接口`DB::Write(const WriteOptions& options, WriteBatch* updates)`调用也是`DBImpl::Write`。

批量写入一个典型问题就是一致性，例如这么调用：

```cpp
leveldb::WriteBatch batch;
batch.Put("company", "Google");
batch.Put(...);
batch.Delete("company");

db->Write(write_option, &batch);
```

我们肯定不希望读到`company -> Google`这个中间结果，而效果的产生就在于`sequence`：`versions_`记录了单调递增的`sequence`，对于相同 key，判断先后顺序依赖该数值。

写入时，`sequence`递增的更新到 memtable，但是一次性的记录到`versions_`:

```cpp
  uint64_t last_sequence = versions_->LastSequence();//本次写入的SequenceNumber
  ...
    WriteBatchInternal::SetSequence(updates, last_sequence + 1);
    last_sequence += WriteBatchInternal::Count(updates);
    ...
    versions_->SetLastSequence(last_sequence);
```

对于`Get`操作(参考本文 Get 一节)，看到的 sequence 只有两种可能：

1. `<= last_sequence`  
2. `>= last_sequence + Count(updates)`  

因此读取时不会观察到中间状态。

## 3. WriteBatch

第一节介绍，写入的`key/value`数据，都记录到了`WriteBatch`，更具体的，记录到了:

```cpp
  //rep_存储了所有Put/Delete接口传入的数据
  //按照一定格式记录了:sequence, count, 操作类型(Put or Delete)，key/value的长度及key/value本身
  std::string rep_;  // See comment in write_batch.cc for the format of rep_
```

`rep_`数据组织如下：

![Write](assets/images/leveldb/WriteBatch.png)

## 4. Get

读取分为 snapshot 读和普通读取，两者的区别只是前面介绍的 sequence 不同。

按照`mem -> imm -> sstable files`的顺序读取，读不到则从下一个介质读取。因此 leveldb 更适合读取最近写入的数据。

```cpp
  // Unlock while reading from files and memtables
  {
    mutex_.Unlock();
    // First look in the memtable, then in the immutable memtable (if any).
    // 查找时需要指定SequenceNumber
    LookupKey lkey(key, snapshot);
    //先查找memtable
    if (mem->Get(lkey, value, &s)) {
      // Done
    //再查找immutable memtable
    } else if (imm != nullptr && imm->Get(lkey, value, &s)) {
      // Done
    } else {
      //查找sstable
      s = current->Get(options, lkey, value, &stats);
      have_stat_update = true;
    }
    mutex_.Lock();
  }
```

`GetStats`则记录了第一个 seek 但是没有查找到 key 的文件，之后[major compaction之筛选文件](https://izualzhy.cn/leveldb-PickCompaction)会用到。

```cpp
  struct GetStats {
    FileMetaData* seek_file;
    int seek_file_level;
  };
```

```cpp
  if (have_stat_update && current->UpdateStats(stats)) {
    MaybeScheduleCompaction();
  }
```

用一张图来表示流程的话：

![Get](assets/images/leveldb/Get.png)
