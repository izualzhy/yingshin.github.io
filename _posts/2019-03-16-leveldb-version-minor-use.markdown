---
title: "leveldb笔记之15:minor compaction 之 version"
date: 2019-03-16 21:45:50
tags: [leveldb]
---

这篇笔记讲下[minor compaction](https://izualzhy.cn/leveldb-compaction)过程中[version](https://izualzhy.cn/leveldb-version)是怎么生效的，算是对 minor compaction 介绍的补充。

`imm_`持久化为 sstable 文件后，文件的相关信息通过`meta`返回

```
  {
    mutex_.Unlock();
    //更新memtable中全部数据到xxx.ldb文件
    //meta记录key range, file_size等sst信息
    s = BuildTable(dbname_, env_, options_, table_cache_, iter, &meta);
    mutex_.Lock();
  }
```

```
struct FileMetaData {
  int refs;
  int allowed_seeks;          // Seeks allowed until compaction
  uint64_t number;
  uint64_t file_size;         // File size in bytes
  InternalKey smallest;       // Smallest internal key served by table
  InternalKey largest;        // Largest internal key served by table

  FileMetaData() : refs(0), allowed_seeks(1 << 30), file_size(0) { }
};
```

`meta`包括文件的 key range，大小，文件的引用数(当引用数为0时会从磁盘删除)等。

查找合适的 level 将新文件记录到`edit`：

```
    //为新生成sstable选择合适的level(不一定总是0)
    if (base != nullptr) {
      level = base->PickLevelForMemTableOutput(min_user_key, max_user_key);
    }
    //level及file meta记录到edit
    edit->AddFile(level, meta.number, meta.file_size,
                  meta.smallest, meta.largest);
```

minor compaction 比较简单，因为只新增了一个 sstable 文件，加入后调用`LogAndApply`生效到新版本。

```
  // Replace immutable memtable with the generated Table
  if (s.ok()) {
    edit.SetPrevLogNumber(0);
    edit.SetLogNumber(logfile_number_);  // Earlier logs no longer needed
    //应用edit
    s = versions_->LogAndApply(&edit, &mutex_);
  }
```
