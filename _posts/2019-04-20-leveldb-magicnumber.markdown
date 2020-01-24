---
title: "leveldb笔记之23:谈谈leveldb的一些魔数"
date: 2019-04-20 18:40:25
tags: [leveldb]
---

代码里经常会有一些奇怪的数字设置，我们称之为魔数(magic number)。这些数字有的没有意义，有的则包含了线上实际经验。很多时候想清楚这些数字设置的原则，对于充分理解一个模块，以及线上优化，都有重要的作用。

这篇笔记，聊聊 leveldb 里的那些数字。

经验不足，有些只是纯粹靠着代码臆测，不对之处还请指正。

## 1. memtable 的最大大小

`option.write_buffer_size`表示一个 memtable 的最大大小，默认为 4M.

## 2. level 0 的文件大小及个数

level 0 的文件是无序的，每次查找 level 0都需要查找所有文件，因此需要严格控制个数。

当文件数目 >= `config::kL0_SlowdownWritesTrigger` 时，通过 sleep 延缓写入，当文件数目 >= `config::kL0_StopWritesTrigger`时，则完全停止写入，等待后台 compaction.这两个数字分别是 8 和 12.

由于 memtable 是通过 `BuildTable` 一次性生成一个文件，因此文件大小 ~ 4M左右.

compact 会有一个 score 的筛选，其中 level 0 的计算公式为：

```cpp
      score = v->files_[level].size() /
          static_cast<double>(config::kL0_CompactionTrigger);
```

当 `score >= 1.0` 时，就会触发往下层的文件归并操作，因此可以认为 level 0 比较理想的个数应当 < `config::kL0_CompactionTrigger`，该值为 4.

## 3. level 1+ 的文件大小及个数

level 最大一共是 7层，范围是`[0, kNumLevels = 7)`.

各 level 的文件都是归并生成的，在`DoCompactionWork`生成文件，当文件大小超过`MaxOutputFileSize`时，则重新打开新文件写入。这个大小对所有 level 都是一样的，默认值是`options->max_file_size = 2M`.

每次的文件总大小是`MaxBytesForLevel`，随着 level 增大，从`10M`到`1T`.

|level  |single file size   |totla file size  |
|--|--|--|
|0  |4M  |4M*12  |
|1  |2M  |10M  |
|2  |2M  |100M  |
|3  |2M  |1G  |
|4  |2M  |10G  |
|5  |2M  |100G  |
|6  |2M  |1T  |

## 4. 文件的重叠

在 major compact 过程中，比如归并 L 层文件到 L + 1，会提前考虑这次 compaction 对 L + 2 的影响，避免 compact 完成后 L + 1 与 L + 2 文件重叠过大。

因此 compact 时，每个 key 都会判断下跟 L + 2 重叠文件的大小，超过`MaxGrandParentOverlapBytes(vset->options_)`时则提前结束本次 compact.

这个值默认大小为 20M，考虑下极端情况:L + 1层才只生成了一个文件，这个文件大小按照上一节的大小最大是 2M。因此一层文件跟下一层最多有 10 个重叠文件。

## 5. table

table 由多个 block 组成，其中每个 data block 最大为 4K，该值定义在`options.block_size`.

`Footer`最后8个字节固定是`0xdb4775248b80fb57ull`，这几个字节的由来是：

```cpp
// kTableMagicNumber was picked by running
//    echo http://code.google.com/p/leveldb/ | sha1sum
// and taking the leading 64 bits.
```

## 6. 缓存的大小

data block 的缓存：总大小默认 8M，单个 data block 为 4K，一共大概能存储 2048 个 block.

leveldb 控制最多占用 1000 个文件句柄，而缓存的 sst 文件`File*`则最多是 9990 个

```cpp
static int TableCacheSize(const Options& sanitized_options) {
  // Reserve ten files or so for other uses and give the rest to TableCache.
  // max_open_files 默认1000
  return sanitized_options.max_open_files - kNumNonTableCacheFiles;
}
```

其中`const int kNumNonTableCacheFiles = 10;`

这 10 个对应了 log manifest current LOCK 等.

## 7. 日志

log 跟 memtable 周期相同，因此也可以近似的认为 ~ 4M

```cpp
     // Attempt to switch to a new memtable and trigger compaction of old
      assert(versions_->PrevLogNumber() == 0);
      uint64_t new_log_number = versions_->NewFileNumber();
      WritableFile* lfile = nullptr;
      s = env_->NewWritableFile(LogFileName(dbname_, new_log_number), &lfile);
      if (!s.ok()) {
        // Avoid chewing through file number space in a tight loop.
        versions_->ReuseFileNumber(new_log_number);
        break;
      }
      delete log_;
      delete logfile_;
      logfile_ = lfile;
      logfile_number_ = new_log_number;
      log_ = new log::Writer(lfile);
      imm_ = mem_;//mem_大小超过4M，因此转化为imm_
      has_imm_.Release_Store(imm_);
      mem_ = new MemTable(internal_comparator_);//重新new一个新的mem_供更新
      mem_->Ref();
```

log 结构由多个 block 组成，每个大小为 32k，这样的好处是，读取时可以固定使用 32k 来不断读取：

```cpp
static const int kBlockSize = 32768;//32k
```
