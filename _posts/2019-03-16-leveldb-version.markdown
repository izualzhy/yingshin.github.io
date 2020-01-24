---
title: "leveldb笔记之14:version"
date: 2019-03-16 12:12:25
tags: [leveldb]
---

[上一篇笔记](https://izualzhy.cn/leveldb-compaction)开始讲 minor compaction，对于 memtable 转化为 sstable 这个过程，在之前的笔记里都做了很多的铺垫，例如 MemTable/SSTable 的数据结构，写入/读取的整个流程等，因此理解起来应该不算复杂。不过其中涉及版本的操作，例如`versions_->LogAndApply(&edit, &mutex_);`，如果没有抓住 leveldb 里 version 相关的关键结构，很容易被绕晕。

这篇笔记，开始讲下 version.

## 1. 为什么要有版本管理

compaction 简言之，是一个新增与删除文件的过程。例如对于上一篇介绍的 minor compaction，是新增一个文件。对于 major compaction，则是归并 N 个文件到 M 个新文件，这 N+M 个历史文件与新文件，共同存储在磁盘上。因此需要一个文件管理系统，能够识别出哪些是当前的 sstable files，哪些属于历史文件。

所以版本管理的作用之一，就是**记录 compaction 之后，数据库由哪些文件组成**

compaction 是 leveldb 单独的线程，当我们读取某个 sstable 文件时，可能该文件正在 compact，也就是作为 N 个历史文件之一。那这个文件虽然不在新的 version 里，但是也不能删除，该文件属于之前的某个 version。

所以版本管理的作用之二，就是**记录文件属于哪个 version**.

一句话，就是**版本管理负责管理磁盘上的文件，以保证 leveldb 数据的准确性**。

## 2. version 的关键结构

从第一节里的介绍的两个作用，版本管理有两个很自然的想法：

### 2.1. delta

每次 compaction 都是新增与删除文件，在原来文件版本的基础上，生成一个新的版本。也就是

```
Version + Delta = New-Version
```

在 leveldb 具体实现中，负责管理 Delta 的类是 `VersionEdit`，某个版本使用`Version`记录。

![Version](/assets/images/leveldb/version-edit.png)

`operator + =`则由类`Builder`实现

### 2.2. 链表

`New-Version`生成后，仅在`Version`下存在的文件并不一定会立刻删除，例如有的文件还在被读取，或者程序直接退出了没有来得及删除。

因此多个 version 会同时存在，之间是链表的关系，当某个 version 彻底没有使用后，其独有的文件才能被删除，同时从链表里删除该 version 。实现这个功能的类是`VersionSet`.

接下来我们具体介绍下每个类的成员变量和接口，通过代码能够看到一个更清晰的一个版本管理系统，以及`Version VersionEdit VersionSet`的关系。

## 3. 源码解析

### 3.1. VersionEdit

`VersionEdit`即 delta，最重要的两个成员变量就是新增与删除文件:

```cpp
  DeletedFileSet deleted_files_;//待删除文件
  //新增文件，例如immutable memtable dump后就会添加到new_files_
  std::vector< std::pair<int, FileMetaData> > new_files_;
```

对外接口上，例如`AddFile`就是更新`new_files_`

```cpp
  // Add the specified file at the specified number.
  // REQUIRES: This version has not been saved (see VersionSet::SaveTo)
  // REQUIRES: "smallest" and "largest" are smallest and largest keys in file
  // 记录{level, FileMetaData}对到new_files_
  void AddFile(int level, uint64_t file,
               uint64_t file_size,
               const InternalKey& smallest,
               const InternalKey& largest) {
    FileMetaData f;
    f.number = file;
    f.file_size = file_size;
    f.smallest = smallest;
    f.largest = largest;
    new_files_.push_back(std::make_pair(level, f));
  }
```

### 3.2. Version

`Version`用于表示某次 compaction 后的数据库状态，管理当前的文件集合，因此最重要一个成员变量`files_`表示每一层的全部 sstable 文件。

```cpp
  // List of files per level
  std::vector<FileMetaData*> files_[config::kNumLevels];
```

#### 3.2.1. PickLevelForMemTableOutput

`PickLevelForMemTableOutput`顾名思义，就是为刚从 memtable 持久化的 sstable，选择一个合适的 level.

选择的几个原则：

1. level 0的 sstable 数量有严格的限制，因此尽可能尝试放到一个更大的 level.  
2. 大于 level 0的各层文件间是有序的，如果放到对应的层数会导致文件间不严格有序，会影响读取，则不再尝试。  
3. 如果放到 level + 1层，与 level + 2层的文件重叠很大，就会导致 compact 到该文件时，压力过大，则不再尝试。这算是一个预测，放到 level 层能够缓冲这一点。  
4. 最大返回 level 2，这大概是个经验值。

```cpp
// 找一个合适的level放置新从memtable dump出的sstable
// 注：不一定总是放到level 0，尽量放到更大的level
// 如果[small, large]与0层有重叠，则直接返回0
// 如果与level + 1文件有重叠，或者与level + 2层文件重叠过大，则都不应该放入level + 1，直接返回level
//返回的level 最大为2
int Version::PickLevelForMemTableOutput(
    const Slice& smallest_user_key,
    const Slice& largest_user_key) {
  //默认放到level 0
  int level = 0;
  if (!OverlapInLevel(0, &smallest_user_key, &largest_user_key)) {
    //如果level 0的文件没有交集
    // Push to next level if there is no overlap in next level,
    // and the #bytes overlapping in the level after that are limited.
    InternalKey start(smallest_user_key, kMaxSequenceNumber, kValueTypeForSeek);
    InternalKey limit(largest_user_key, 0, static_cast<ValueType>(0));
    std::vector<FileMetaData*> overlaps;
    while (level < config::kMaxMemCompactLevel) {//kMaxMemCompactLevel = 2，因此level = 0 or 1
      //与level + 1(下一层)文件有交集，只能直接返回该层
      //目的是为了保证下一层文件是有序的
      if (OverlapInLevel(level + 1, &smallest_user_key, &largest_user_key)) {
        break;
      }
      if (level + 2 < config::kNumLevels) {
        //如果level + 2(下两层)的文件与key range有重叠的文件大小超过20M
        //目的是避免放入level + 1层后，与level + 2 compact时文件过大
        // Check that file does not overlap too many grandparent bytes.
        GetOverlappingInputs(level + 2, &start, &limit, &overlaps);
        const int64_t sum = TotalFileSize(overlaps);
        if (sum > MaxGrandParentOverlapBytes(vset_->options_)) {
          break;
        }
      }
      level++;
    }
  }
  return level;
    }
```

### 3.3. Builder

`Builder`是一个辅助类，实现`Version + VersionEdit = Version'`的功能，其中`+ =`分别对应`Apply SaveTo`两个接口。

![VersionSet::Builder](/assets/images/leveldb/version-set-builder.png)

成员变量也是记录所有的 delta，`levels_`存储了每一层的`added_files`及`deleted_files`:

```cpp
  typedef std::set<FileMetaData*, BySmallestKey> FileSet;
  struct LevelState {
    std::set<uint64_t> deleted_files;
    FileSet* added_files;
  };

  VersionSet* vset_;
  Version* base_;
  LevelState levels_[config::kNumLevels];//每一层的新增及删除文件
```


#### 3.3.1. Apply

`Apply`主要就是更新新增及删除文件的集合，首先是删除文件：

```cpp
  // 记录edit中可删除及新增文件到levels_
  void Apply(VersionEdit* edit) {
    // Update compaction pointers
    // 下次compact的起始key
    for (size_t i = 0; i < edit->compact_pointers_.size(); i++) {
      const int level = edit->compact_pointers_[i].first;
      vset_->compact_pointer_[level] =
          edit->compact_pointers_[i].second.Encode().ToString();
    }

    // Delete files
    // 记录可删除文件到各level对应的deleted_files
    const VersionEdit::DeletedFileSet& del = edit->deleted_files_;
    for (VersionEdit::DeletedFileSet::const_iterator iter = del.begin();
         iter != del.end();
         ++iter) {
      const int level = iter->first;
      const uint64_t number = iter->second;
      levels_[level].deleted_files.insert(number);
    }
```

从`edit`里读取所有的待删除文件，更新到对应 level 的`deleted_files`.  
注：`compact_pointers_`主要用于 major compact 时选择文件。  

接下来的执行则是更新对应 level 的`added_files`：

```cpp
    // Add new files
    // 记录新增文件到added_files，并计算该文件的allowed_seeks(用于触发compact)
    for (size_t i = 0; i < edit->new_files_.size(); i++) {
      const int level = edit->new_files_[i].first;
      FileMetaData* f = new FileMetaData(edit->new_files_[i].second);
      f->refs = 1;
      ...
      levels_[level].deleted_files.erase(f->number);
      levels_[level].added_files->insert(f);
    }
```

注：这里有一个`allowed_seeks`的计算，用于 major compact 时选择文件，参考[seek_compaction](https://izualzhy.cn/leveldb-version-for-compaction#1-seek_compaction)

经过`Apply`后，`levels_`更新完成。

#### 3.3.2. SaveTo

`SaveTo`就是将`levels_`记录的 deleted/added files 作用于原来的，生成新的version v，就是遍历每一层，将 delta 的文件更新到合适位置。

```cpp
  // Save the current state in *v.
  // 将levels_记录的deleted/added files作用于base，生成新的version v
  void SaveTo(Version* v) {
    BySmallestKey cmp;
    cmp.internal_comparator = &vset_->icmp_;
    for (int level = 0; level < config::kNumLevels; level++) {
      // Merge the set of added files with the set of pre-existing files.
      // Drop any deleted files.  Store the result in *v.
      // 当前level的原有文件
      const std::vector<FileMetaData*>& base_files = base_->files_[level];
      std::vector<FileMetaData*>::const_iterator base_iter = base_files.begin();
      std::vector<FileMetaData*>::const_iterator base_end = base_files.end();
      // edit里的新增文件
      const FileSet* added = levels_[level].added_files;
      // 新的version vector预留空间，防止频繁copy
      v->files_[level].reserve(base_files.size() + added->size());
      // 遍历所有新增文件，按照顺序把base_files added_files有序加到v->files_
      for (FileSet::const_iterator added_iter = added->begin();
           added_iter != added->end();
           ++added_iter) {
        // Add all smaller files listed in base_
        // 按照BySmallestKey排序，找到原有文件中比added_iter小的文件，加入到v
        for (std::vector<FileMetaData*>::const_iterator bpos
                 = std::upper_bound(base_iter, base_end, *added_iter, cmp);
             base_iter != bpos;
             ++base_iter) {
          MaybeAddFile(v, level, *base_iter);
        }

        //接着把added_iter加入到v，这样保证了v里文件的顺序
        MaybeAddFile(v, level, *added_iter);
      }

      // Add remaining base files
      for (; base_iter != base_end; ++base_iter) {
        MaybeAddFile(v, level, *base_iter);
      }
    }
  }
```

### 3.4. VersionSet

随着`Builder`不断执行，新的`version`被构造出来。`VersionSet`就负责管理多个版本，对应的变量全局唯一，在`DBImpl`构造函数里初始化：

```cpp
      versions_(new VersionSet(dbname_, &options_, table_cache_,
                               &internal_comparator_)) {
```

管理一个双向链表

```cpp
  Version dummy_versions_;  // Head of circular doubly-linked list of versions.
  Version* current_;        // == dummy_versions_.prev_
```

![VersionSet](/assets/images/leveldb/version-set.png)

`current_`指向最新的版本。

因此`class Version`实际上还有三个重要的链表相关成员变量：

```cpp
  VersionSet* vset_;            // VersionSet to which this Version belongs
  Version* next_;               // Next version in linked list
  Version* prev_;               // Previous version in linked list
```

#### 3.4.1 LogAndApply

`Status VersionSet::LogAndApply(VersionEdit* edit, port::Mutex* mu)`的主要做了几件事：

1. 将`edit`应用于`current_`生成一个新的`Version`  
2. 计算新`Version`下，下次 major compact 的文件  
3. 更新一些元信息管理文件  
4. 将新`Version`添加到双向链表，`current_ = 新Version`

首先是生成新`Version`:

```cpp
  Version* v = new Version(this);
  {
    Builder builder(this, current_);
    builder.Apply(edit);
    builder.SaveTo(v);
  }
```

接着调用`Finalize`计算下次 major compact 时要处理的层，参考[size_compaction](https://izualzhy.cn/leveldb-version-for-compaction#2-size_compaction)

```cpp
  Finalize(v);
```

更新`manifest`写入`current_`

```cpp
  // Initialize new descriptor log file if necessary by creating
  // a temporary file that contains a snapshot of the current version.
  std::string new_manifest_file;
  Status s;
  if (descriptor_log_ == nullptr) {
    // No reason to unlock *mu here since we only hit this path in the
    // first call to LogAndApply (when opening the database).
    assert(descriptor_file_ == nullptr);
    //形如MANIFEST-xxxxxx的文件名
    new_manifest_file = DescriptorFileName(dbname_, manifest_file_number_);
    edit->SetNextFile(next_file_number_);
    s = env_->NewWritableFile(new_manifest_file, &descriptor_file_);
    if (s.ok()) {
      descriptor_log_ = new log::Writer(descriptor_file_);
      // manifest写入current_的信息
      s = WriteSnapshot(descriptor_log_);
    }
  }
```

写入`edit`

```cpp
  // Unlock during expensive MANIFEST log write
  {
    mu->Unlock();

    // Write new record to MANIFEST log
    if (s.ok()) {
      std::string record;
      edit->EncodeTo(&record);
      // manifest写入本次edit的信息
      s = descriptor_log_->AddRecord(record);
      if (s.ok()) {
        s = descriptor_file_->Sync();
      }
      if (!s.ok()) {
        Log(options_->info_log, "MANIFEST write: %s\n", s.ToString().c_str());
      }
    }
```

`manifest`就更新完成了，注意格式跟[log](https://izualzhy.cn/leveldb-log)相同。

接着在`CURRENT`文件里明文写入`manifest`文件名。

```cpp
    // 将manifest_file_number_写入CURRENT文件
    if (s.ok() && !new_manifest_file.empty()) {
      s = SetCurrentFile(env_, dbname_, manifest_file_number_);
    }
```

这两个文件在 leveldb 数据库文件里都能找到，形如`MANIFEST-000004 CURRENT`.

最后就是调用`AppendVersion(v);`将新版本更新到链表，修改`current_`：

```cpp
// v加到链表里
void VersionSet::AppendVersion(Version* v) {
  // Make "v" current
  assert(v->refs_ == 0);
  assert(v != current_);
  if (current_ != nullptr) {
    current_->Unref();
  }
  current_ = v;
  v->Ref();

  // Append to linked list
  v->prev_ = dummy_versions_.prev_;
  v->next_ = &dummy_versions_;
  v->prev_->next_ = v;
  v->next_->prev_ = v;
}
```

这样，就完成了将`edit`生效的全部过程。

对应磁盘上的文件就是这个样子：

![manifest](/assets/images/leveldb/manifest.png)

#### 3.4.2. PickCompaction

## 4. 例子

### 4.1. 读取 MANIFEST 文件

MANIFEST 文件是跟日志格式一样，因此，我们按照读取日志的方式读取该文件。查看打开一个 db 下的文件

```cpp
int main() {
    leveldb::SequentialFile* file;
    //MANIFEST files
    leveldb::Status status = leveldb::Env::Default()->NewSequentialFile("./data/test_table.db/MANIFEST-000004", &file);
    std::cout << status.ToString() << std::endl;

    leveldb::log::Reader reader(file, NULL, true/*checksum*/, 0/*initial_offset*/);
    // Read all the records and add to a memtable
    std::string scratch;
    leveldb::Slice record;
    while (reader.ReadRecord(&record, &scratch) && status.ok()) {
        leveldb::VersionEdit edit;
        edit.DecodeFrom(record);
        std::cout << edit.DebugString() << std::endl;
    }
}
```

输出会类似这个样子，也就是对应写入多个`VersionEdit`的过程：

```
OK
28
VersionEdit {
  Comparator: leveldb.BytewiseComparator
}

42
VersionEdit {
  LogNumber: 6
  PrevLogNumber: 0
  NextFile: 7
  LastSeq: 3
  AddFile: 0 5 172 'company' @ 2 : 1 .. 'name' @ 1 : 1
}
```
