---
title: "leveldb笔记之16:seek_compaction && size_compaction"
date: 2019-03-16 22:12:24
tags: [leveldb]
---

major compaction 一个首要问题，就是要 compact 哪个文件。这个计算是在[version](https://izualzhy.cn/leveldb-version#331-apply)完成的，与之相关的主要是两个属性`allowed_seeks compaction_score`。

## 1. seek_compaction

一个直观的想法是如果文件多次 seek 但是没有查找到数据，那么就应该被 compact 了，否则会浪费更多的 seek。用一次 compact 来解决长久空 seek 的问题，本质上，还是如何平衡读写的思想。

具体的，当一个新文件更新进入版本管理，计算该文件允许 seek 但是没有查到数据的最大次数，当超过该次数后，就应该 compact 该文件了。

对应到代码实现，是在`VersionSet::Builder::Apply`阶段:<https://izualzhy.cn/leveldb-version#331-apply>

主要用到的变量是`allowed_seeks`

```cpp
      // We arrange to automatically compact this file after
      // a certain number of seeks.  Let's assume:
      //   (1) One seek costs 10ms
      //   (2) Writing or reading 1MB costs 10ms (100MB/s)
      //   (3) A compaction of 1MB does 25MB of IO:
      //         1MB read from this level
      //         10-12MB read from next level (boundaries may be misaligned)
      //         10-12MB written to next level
      // This implies that 25 seeks cost the same as the compaction
      // of 1MB of data.  I.e., one seek costs approximately the
      // same as the compaction of 40KB of data.  We are a little
      // conservative and allow approximately one seek for every 16KB
      // of data before triggering a compaction.
      //
      // 1. 1次seek花费10ms
      // 2. 1M读写花费10ms
      // 3. 1M文件的compact需要25M IO(读写10-12MB的下一层文件)，为什么10-12M?经验值?
      // 因此1M的compact时间 = 25次seek时间 = 250ms
      // 也就是40K的compact时间 = 1次seek时间，保守点取16KB，即t = 16K的compact时间 = 1次seek时间
      // compact这个文件的时间: file_size / 16K
      // 如果文件seek很多次但是没有找到key，时间和已经比compact时间要大，就应该compact了
      // 这个次数记录到f->allowed_seeks
      f->allowed_seeks = (f->file_size / 16384);//16KB
      if (f->allowed_seeks < 100) f->allowed_seeks = 100;
```

当查找文件而没有查找到时，`allowed_seeks--`，降为0时该文件标记到`file_to_compact_`：

```cpp
bool Version::UpdateStats(const GetStats& stats) {
  FileMetaData* f = stats.seek_file;
  if (f != nullptr) {
    f->allowed_seeks--;
    //如果allowed_seeks降到0，则记录该文件需要compact了
    if (f->allowed_seeks <= 0 && file_to_compact_ == nullptr) {
      file_to_compact_ = f;
      file_to_compact_level_ = stats.seek_file_level;
      return true;
    }
  }
  return false;
}
```

## 2. size_compaction

compaction 另外一个直观的想法就是，当某一层文件变得很大，往往意味着冗余数据过多，应该 compact 以避免占用磁盘以及读取过慢的问题。

level 越大，我们可以认为数据越“冷”，读取的几率越小，因此大的 level，能“容忍”的程度就越高，给的文件大小阈值越大。

具体的，当产生新版本时，遍历所有的层，比较该层文件总大小与基准大小，得到一个最应当 compact 的层。

这个步骤，在[VersionSet::Finalize](https://izualzhy.cn/leveldb-version#341-logandapply)完成。

```cpp
//计算compact的level和score，更新到compaction_level_&&compaction_score_
void VersionSet::Finalize(Version* v) {
  // Precomputed best level for next compaction
  int best_level = -1;
  double best_score = -1;

  //level 0看文件个数，降低seek的次数，提高读性能，个数/4
  //level >0看文件大小，减少磁盘占用，大小/(10M**level)
  //例如:
  //level 0 有4个文件，score = 1.0
  //level 1 文件大小为9M，score = 0.9
  //那么compact的level就是0,score = 1.0
  for (int level = 0; level < config::kNumLevels-1; level++) {
    double score;
    if (level == 0) {
      // We treat level-0 specially by bounding the number of files
      // instead of number of bytes for two reasons:
      //
      // (1) With larger write-buffer sizes, it is nice not to do too
      // many level-0 compactions.
      //
      // (2) The files in level-0 are merged on every read and
      // therefore we wish to avoid too many files when the individual
      // file size is small (perhaps because of a small write-buffer
      // setting, or very high compression ratios, or lots of
      // overwrites/deletions).
      score = v->files_[level].size() /
          static_cast<double>(config::kL0_CompactionTrigger);
    } else {
      // Compute the ratio of current size to size limit.
      const uint64_t level_bytes = TotalFileSize(v->files_[level]);
      score =
          static_cast<double>(level_bytes) / MaxBytesForLevel(options_, level);
    }

    if (score > best_score) {
      best_level = level;
      best_score = score;
    }
  }

  v->compaction_level_ = best_level;
  v->compaction_score_ = best_score;
}

//10**level，即level 1 = 10M, level 2 = 100M, ...                                                                  static double MaxBytesForLevel(const Options* options, int level) {
  // Note: the result for level zero is not really used since we set
  // the level-0 compaction threshold based on number of files.

  // Result for both level-0 and level-1
  double result = 10. * 1048576.0;//10M
  while (level > 1) {
    result *= 10;
    level--;
  }
  return result;
}
```

可以看到 level 0 与其他层不同，看的是文件个数，因为 level 0 的文件是重叠的，每次读取都需要遍历所有文件，所以文件个数更加影响性能。

每层的基准大小为`10M << ${level - 1}`，level = 1 则`MaxBytes = 10M`, level = 2 则`MaxBytes=100M`，依次类推.

逐层比较后，得到最大的得分以及对应层数：`compaction_score_ compaction_level_`

major compact 选择文件时就会用到上述两个条件。

```cpp
  // We prefer compactions triggered by too much data in a level over
  // the compactions triggered by seeks.
  const bool size_compaction = (current_->compaction_score_ >= 1);//文件数过多
  const bool seek_compaction = (current_->file_to_compact_ != nullptr);//seek了多次文件但是没有查到，记录到的file_to_compact_
```

因此，[版本管理](https://izualzhy.cn/leveldb-version#1-%E4%B8%BA%E4%BB%80%E4%B9%88%E8%A6%81%E6%9C%89%E7%89%88%E6%9C%AC%E7%AE%A1%E7%90%86)的作用之三，就是在新增或者遍历文件的过程中，为 major compact 筛选文件。
