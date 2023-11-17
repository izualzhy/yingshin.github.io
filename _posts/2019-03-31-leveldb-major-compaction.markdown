---
title: "leveldb笔记之19:major compaction"
date: 2019-03-31 12:21:23
tags: leveldb
---

[minor compaction](https://izualzhy.cn/leveldb-compaction)主要解决内存数据持久化磁盘。major compaction 则负责将磁盘上的数据合并，每合并一次，数据就落到更底一层。

## 1. 作用

随着数据合并到更大的 level，一个明显的好处就是清理冗余数据。

如果不同 level 的 sst 文件里，存在相同的 key，那么更底层的数据就可以删除不再保留（不考虑 snapshot的情况下）。为了充分利用磁盘高性能的顺序写，删除数据也是顺序写入删除标记，而真正删除数据，是在 major compact 的过程中。

所以，一个作用是能够节省磁盘空间。

level 0 的数据文件之间是无序的，每次查找都需要遍历所有可能重叠的文件，而归并到 level 1 之后，数据变得有序，待查找的文件变少。

所以，另外一个作用是能够提高读效率。

## 2. 源码解析

compact 的 完整操作在`DBImpl::BackgroundCompaction`实现，包含了 minor && major compaction.

### 2.1. BackgroundCompaction

首先是查看是否需要 compact memetable.

```cpp
//实际Compact
void DBImpl::BackgroundCompaction() {
  mutex_.AssertHeld();

  //如果immutable memtable存在，则本次先compact，即Minor Compaction
  if (imm_ != nullptr) {
    CompactMemTable();
    return;
  }
```

接着就是调用[PickCompaction](https://izualzhy.cn/leveldb-PickCompaction)筛选合适的 level 及 文件。注意也可以手动指定 range，原理是类似的，不再赘述。

```cpp
  //如果immutable memtable不存在，则合并各层level的文件，称为Major Compaction
  Compaction* c;
  bool is_manual = (manual_compaction_ != nullptr);
  InternalKey manual_end;
  //手动指定compact
  if (is_manual) {
    ManualCompaction* m = manual_compaction_;
    c = versions_->CompactRange(m->level, m->begin, m->end);
    m->done = (c == nullptr);
    if (c != nullptr) {
      manual_end = c->input(0, c->num_input_files(0) - 1)->largest;
    }
    Log(options_.info_log,
        "Manual compaction at level-%d from %s .. %s; will stop at %s\n",
        m->level,
        (m->begin ? m->begin->DebugString().c_str() : "(begin)"),
        (m->end ? m->end->DebugString().c_str() : "(end)"),
        (m->done ? "(end)" : manual_end.DebugString().c_str()));
  } else {
  //自动compact，c记录了待参与compact的所有文件
    c = versions_->PickCompaction();
  }
```

### 2.2. TrivialMove

返回`Compaction*`后，首先是一个比较取巧的设计：

**什么条件下，可以直接使用原文件，而节省重新生成文件这个过程？**

那就是`IsTrivialMove`:

```cpp
bool Compaction::IsTrivialMove() const {
  const VersionSet* vset = input_version_->vset_;
  // Avoid a move if there is lots of overlapping grandparent data.
  // Otherwise, the move could create a parent file that will require
  // a very expensive merge later on.
  // 同时满足以下条件时，我们只要简单的把文件从level标记到level + 1层就可以了
  // 1. level层只有一个文件
  // 2. level + 1层没有文件
  // 3. 跟level + 2层overlap的文件没有超过25M
  // 注：条件三主要是(避免mv到level + 1后，导致level + 1 与 level + 2层compact压力过大)
  return (num_input_files(0) == 1 && num_input_files(1) == 0 &&
          TotalFileSize(grandparents_) <=
              MaxGrandParentOverlapBytes(vset->options_));
}
```

由于跟 level + 1 层文件没有重叠，直接 mv 到下一层并不会导致错误。

```cpp
  Status status;
  if (c == nullptr) {
    // Nothing to do
  } else if (!is_manual && c->IsTrivialMove()) {
    // Move file to next level
    // level + 1没有overlap的文件，不需要compact，直接从level层标记到level + 1层即可
    assert(c->num_input_files(0) == 1);
    FileMetaData* f = c->input(0, 0);
    //直接把这个文件从level移动level + 1层
    c->edit()->DeleteFile(c->level(), f->number);
    c->edit()->AddFile(c->level() + 1, f->number, f->file_size,
                       f->smallest, f->largest);
    status = versions_->LogAndApply(c->edit(), &mutex_);
    if (!status.ok()) {
      RecordBackgroundError(status);
    }
    VersionSet::LevelSummaryStorage tmp;
    Log(options_.info_log, "Moved #%lld to level-%d %lld bytes %s: %s\n",
        static_cast<unsigned long long>(f->number),
        c->level() + 1,
        static_cast<unsigned long long>(f->file_size),
        status.ToString().c_str(),
        versions_->LevelSummary(&tmp));
```

可以看到如果满足 TrivialMove 的条件，只要在 edit 里记录下然后通过`LogAndApply`生效就可以了。因此没有操作 sst 文件，只是修改文件对应的 level.

### 2.3. DoCompactionWork

正常情况下，通过`DoCompactionWork`完成文件的归并操作。

```cpp
  } else {
    CompactionState* compact = new CompactionState(c);
    status = DoCompactionWork(compact);
    if (!status.ok()) {
      RecordBackgroundError(status);
    }
    CleanupCompaction(compact);
    c->ReleaseInputs();
    DeleteObsoleteFiles();//删除旧文件，回收内存和磁盘空间
  }
```

```cpp
//真正的compaction，compact里记录了本次所有参与compact的文件
Status DBImpl::DoCompactionWork(CompactionState* compact) {
```

实现上，主要就是通过遍历所有的文件，实现多路归并，生成新的文件。

第一步，获取遍历所有文件用到的 `Iterator*`.

```cpp
  //input用于遍历compact里所有文件的key
  Iterator* input = versions_->MakeInputIterator(compact->compaction);
  input->SeekToFirst();
```

#### 2.3.1. 生成遍历的 iterator

`MakeInputIterator`返回的是[MergeIterator](https://izualzhy.cn/leveldb-iterator#4-mergingiterator)，包含了对`c->inputs_`两层文件的迭代器。

```cpp
  // level 0：文件是无序的，有多少个sstable file，就需要多少Iterator
  // level >0：文件是有序的，1个Iterator就可以了
  const int space = (c->level() == 0 ? c->inputs_[0].size() + 1 : 2);
  // list存储所有Iterator
  Iterator** list = new Iterator*[space];
  int num = 0;
  for (int which = 0; which < 2; which++) {
    if (!c->inputs_[which].empty()) {
      //第0层
      if (c->level() + which == 0) {
        const std::vector<FileMetaData*>& files = c->inputs_[which];
        // Iterator* Table::NewIterator
        for (size_t i = 0; i < files.size(); i++) {
          list[num++] = table_cache_->NewIterator(
              options, files[i]->number, files[i]->file_size);
        }
      } else {
        // Create concatenating iterator for the files from this level
        list[num++] = NewTwoLevelIterator(
            // 遍历文件列表的iterator
            new Version::LevelFileNumIterator(icmp_, &c->inputs_[which]),
            &GetFileIterator, table_cache_, options);
      }
    }
  }
```

#### 2.3.2. 遍历

iterator 返回的 key 全部有序，遍历过程可以清理掉一些 key。

由于多次Put/Delete，有些key会出现多次，在compact时丢弃。策略如下：  
1. 对于多次出现的user key，我们只关心最后写入的值 or >snapshot的值通过设置last_sequence_for_key = kMaxSequenceNumber以及跟compact->smallest_snapshot比较，可以分别保证这两点  
2. 如果是删除key && <= snapshot && 更高层没有该key，那么也可以忽略  

同时跟上一节的思想类似，如果目前 compact 生成的文件，会导致接下来 level + 1 && level + 2 层 compact 压力过大，那么结束本次 compact.

因此，每次都会调用`ShouldStopBefore`来判断是否满足上述条件：

```cpp
  //从小到大遍历
  for (; input->Valid() && !shutting_down_.Acquire_Load(); ) {
    // Prioritize immutable compaction work
    if (has_imm_.NoBarrier_Load() != nullptr) {
      const uint64_t imm_start = env_->NowMicros();
      mutex_.Lock();
      if (imm_ != nullptr) {
        CompactMemTable();
        // Wake up MakeRoomForWrite() if necessary.
        background_work_finished_signal_.SignalAll();
      }
      mutex_.Unlock();
      imm_micros += (env_->NowMicros() - imm_start);
    }

    Slice key = input->key();
    //与level + 2层的文件比较，如果目前的compact已经会导致后续level + 1 与 level + 2 compact压力过大
    //那么结束本次compact
    if (compact->compaction->ShouldStopBefore(key) &&
        compact->builder != nullptr) {
      status = FinishCompactionOutputFile(compact, input);
      if (!status.ok()) {
        break;
      }
    }
```

`drop`用于判断数据是否可以丢弃：

```cpp
    // Handle key/value, add to state, etc.
    bool drop = false;
    if (!ParseInternalKey(key, &ikey)) {
      // Do not hide error keys
      current_user_key.clear();
      has_current_user_key = false;
      last_sequence_for_key = kMaxSequenceNumber;
    } else {
      if (!has_current_user_key ||
          user_comparator()->Compare(ikey.user_key,
                                     Slice(current_user_key)) != 0) {
        // First occurrence of this user key
        // 相同user key可能会有多个，seq越大表示越新，顺序越靠前
        // 第一次碰到该user_key，标记has_current_user_key为true, sequence为max值
        current_user_key.assign(ikey.user_key.data(), ikey.user_key.size());
        has_current_user_key = true;
        last_sequence_for_key = kMaxSequenceNumber;
      }

      //由于多次Put/Delete，有些key会出现多次
      //有几种情况可以忽略key，在compact时丢弃：
      //1. 对于多次出现的user key，我们只关心最后写入的值 or >snapshot的值
      //   通过设置last_sequence_for_key = kMaxSequenceNumber
      //   以及跟compact->smallest_snapshot比较，可以分别保证这两点
      //2. 如果是删除key && <= snapshot && 更高层没有该key，那么也可以忽略
      if (last_sequence_for_key <= compact->smallest_snapshot) {
        // Hidden by an newer entry for same user key
        drop = true;    // (A)
      } else if (ikey.type == kTypeDeletion &&
                 ikey.sequence <= compact->smallest_snapshot &&
                 compact->compaction->IsBaseLevelForKey(ikey.user_key)) {
        // For this user key:
        // (1) there is no data in higher levels
        // (2) data in lower levels will have larger sequence numbers
        // (3) data in layers that are being compacted here and have
        //     smaller sequence numbers will be dropped in the next
        //     few iterations of this loop (by rule (A) above).
        // Therefore this deletion marker is obsolete and can be dropped.
        drop = true;
      }

      last_sequence_for_key = ikey.sequence;//更新为真正的SequenceNumber
    }
```

如果`drop = false`，说明 key 需要保留，写入新的文件。写入跟 memetable->sstable 一样，通过[TableBuilder](https://izualzhy.cn/leveldb-sstable#4-class-leveldbtablebuilder)完成。

```cpp
    if (!drop) {
      // Open output file if necessary
      // 如果builder为空，则打开文件构造builder，用于数据写入
      if (compact->builder == nullptr) {
        status = OpenCompactionOutputFile(compact);
        if (!status.ok()) {
          break;
        }
      }
      if (compact->builder->NumEntries() == 0) {
        compact->current_output()->smallest.DecodeFrom(key);
      }
      compact->current_output()->largest.DecodeFrom(key);
      compact->builder->Add(key, input->value());//写入本次数据

      // Close output file if it is big enough
      if (compact->builder->FileSize() >=
          compact->compaction->MaxOutputFileSize()) {
        //compact的文件超过了大小(默认2M)，则关闭当前打开的sstable，持久化到磁盘。
        status = FinishCompactionOutputFile(compact, input);
        if (!status.ok()) {
          break;
        }
      }
```

最后通过`InstallCompactionResults`将结果记录到 version，本质上也还是调用的我们之前介绍过的[LogAndApply](https://izualzhy.cn/leveldb-version#341-logandapply).

```cpp
Status DBImpl::InstallCompactionResults(CompactionState* compact) {
  mutex_.AssertHeld();
  Log(options_.info_log,  "Compacted %d@%d + %d@%d files => %lld bytes",
      compact->compaction->num_input_files(0),
      compact->compaction->level(),
      compact->compaction->num_input_files(1),
      compact->compaction->level() + 1,
      static_cast<long long>(compact->total_bytes));

  // Add compaction outputs
  compact->compaction->AddInputDeletions(compact->compaction->edit());
  const int level = compact->compaction->level();
  for (size_t i = 0; i < compact->outputs.size(); i++) {
    const CompactionState::Output& out = compact->outputs[i];
    //新生成的文件增加到edit
    compact->compaction->edit()->AddFile(
        level + 1,
        out.number, out.file_size, out.smallest, out.largest);
  }
  return versions_->LogAndApply(compact->compaction->edit(), &mutex_);
}
```
