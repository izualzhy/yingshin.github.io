---
title: "leveldb笔记之13:minor compaction"
date: 2019-03-09 22:15:24
tags: [leveldb]
---

在[level笔记开篇](https://izualzhy.cn/leveldb-architecture)里，提到过**Compaction**这个过程，这是 leveldb 中最为复杂的一部分，从这篇笔记开始，介绍下 Compaction.

compaction 分为两种:minor compaction 和 major compaction. minor compaction 相对简单一些，对应了[MemTable](https://izualzhy.cn/memtable-leveldb)持久化为[sstable](https://izualzhy.cn/leveldb-sstable)的过程。

## 1. 简介

[DBImpl](https://github.com/yingshin/leveldb_more_annotation/blob/master/db/db_impl.h)定义了两个`MemTable`:

```
  MemTable* mem_;
  MemTable* imm_ GUARDED_BY(mutex_);  // Memtable being compacted
```


当`size of mem_ <= options_.write_buffer_size`时，数据都会直接更新到`mem_`。超过大小后，`mem_`转化为`imm_`：

```
      imm_ = mem_;//mem_大小超过4M，因此转化为imm_
      has_imm_.Release_Store(imm_);
      mem_ = new MemTable(internal_comparator_);//重新new一个新的mem_供更新
```

`imm_`全称是 immutable memtable，只读状态，leveldb 会有一个后台线程负责将`imm_`持久化到磁盘，成为 level 0 的 sst 文件，同时更新一些版本信息（版本的概念会在另外的笔记介绍）、文件清理等。

整个过程完成后，就可以重新设置`imm_ = nullptr;`，当`mem_`大小再次达到阈值，循环这个过程。

如果 minor compaction 耗时较长，会直接导致`mem_`过大无法写入但是又无法转化为`imm_`，因此对 minor compaction 最重要的原则是：

**高性能的持久化**

## 2. 源码

整个过程的入口函数

```
//实际Compact
void DBImpl::BackgroundCompaction() {
  mutex_.AssertHeld();

  //如果immutable memtable存在，则本次先compact，即Minor Compaction
  if (imm_ != nullptr) {
    CompactMemTable();
    return;
  }

  ...
  // major compaction
```

可见 minor compaction 是高优于 major 的。

### 2.1. 整体流程

`CompactMemTable`主要流程分为三部分：

1. `WriteLevel0Table(imm_, &edit, base)`：`imm_`落盘成为新的 sst 文件，文件信息记录到 `edit`  
2. `versions_->LogAndApply(&edit, &mutex_)`：将本次文件更新信息`versions_`，当前的文件（包含新的 sst 文件）作为数据库的一个最新状态，后续读写都会基于该状态  
3. `DeleteObsoleeteFiles`：删除一些无用文件  

### 2.2. `WriteLevel0Table`

首先顺序生成 sstable 的编号，用于文件名  

```
meta.number = versions_->NewFileNumber()
```

`iter`用过遍历 MemTable，通过`BuildTable`将数据写入到 sstable，该函数实际上就是调用了[TableBuilder](https://izualzhy.cn/leveldb-sstable#4-class-leveldbtablebuilder).

```
    mutex_.Unlock();
    //更新memtable中全部数据到xxx.ldb文件
    //meta记录key range, file_size等sst信息
    s = BuildTable(dbname_, env_, options_, table_cache_, iter, &meta);
```

接下来通过`meta`选取合适的 level，注意虽然函数名字是`WriteLevel0Table`，但是新生成 sstable，并不一定总是会放到 level 0，例如如果 key range 与 level 1层的所有文件都没有 overlap，那就会直接放到 level 1。`PickLevelForMemTableOutput`是`Version`的接口，后续笔记专门介绍 leveldb 的版本，这里的作用就是返回一个该 sstable 即将放入的 level，加上`meta`里的文件信息，统一记录到`edit`.

```
  int level = 0;
  if (s.ok() && meta.file_size > 0) {
    const Slice min_user_key = meta.smallest.user_key();
    const Slice max_user_key = meta.largest.user_key();
    //为新生成sstable选择合适的level(不一定总是0)
    if (base != nullptr) {
      level = base->PickLevelForMemTableOutput(min_user_key, max_user_key);
    }
    //level及file meta记录到edit
    edit->AddFile(level, meta.number, meta.file_size,
                  meta.smallest, meta.largest);
  }
```

### 2.3. `LogAndApply`

将`edit`应用到版本信息里记录

```
  // Replace immutable memtable with the generated Table
  if (s.ok()) {
    edit.SetPrevLogNumber(0);
    edit.SetLogNumber(logfile_number_);  // Earlier logs no longer needed
    //应用edit
    s = versions_->LogAndApply(&edit, &mutex_);
  }
```

### 2.4. `DeleteObsoleteFiels`

新的版本信息生成后，会有一些不再需要的文件，通过`DeleteObsoleteFiles`选出并且删除。

```
  if (s.ok()) {
    // Commit to the new state
    imm_->Unref();
    imm_ = nullptr;
    has_imm_.Release_Store(nullptr);
    DeleteObsoleteFiles();
  } else {
    RecordBackgroundError(s);
  }
```

这就是一个完整的 minor compaction过程，通过 minor compaction，内存里的 memtable 源源不断的转化为磁盘上的 sst 文件，正如前面强调的这一过程最重要的原则，就是高性能的写转化。因此并没有考虑不同文件间数据的重复和顺序，当读取数据时，总是需要读取 level 0 所有的文件，因此对读并不友好。major compaction就是为了解决这一问题，不断的把上层的 sst 文件通过二路归并转化为下层的 sst 文件。

从 compact 里，就开始引入 leveldb 里版本的概念了，minor compaction 过程相对简单一些，因此，接下来一篇笔记，开始从上面提到的版本的各个接口，介绍 leveldb 里的版本的设计思想。
