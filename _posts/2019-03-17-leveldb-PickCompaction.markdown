---
title: "leveldb笔记之17:major compaction之筛选文件"
date: 2019-03-23 12:25:51
tags: [leveldb]
---

继续接着[上一篇笔记](https://izualzhy.cn/leveldb-version-for-compaction)的结尾讲，这篇笔记介绍下筛选参与 major compaction 文件的过程，算是介绍 major compaction 的一个开端。

## 1. 当我们谈论筛选时，在谈论什么

leveldb 最为复杂的在 compaction，compaction 最为复杂的在 major compaction.面对磁盘上的众多 sstable 文件，应该怎么开始？

千里之行始于足下，首先需要找到**最应该 compact 的一个文件**。

“最应该”的判断条件，前面笔记已有介绍，有`seek_compaction && size_compaction`，分别从读取和文件大小两个维度来判断。

筛选出这个文件后，还需要考虑一系列问题：

1. 如果在 level 0，由于该层文件之间是无序的，如果只把这一个文件 compact 到 level 1 是否会导致读取错误？  
2. compact 到 level + 1后，会不会导致 level + 1 与 level + 2 的 compact 过于复杂？  
3. 这个文件应该与哪些文件 compact?

这些问题，都需要在`PickCompaction`这个函数里解决。

## 2. `Compaction`

`leveldb::Compaction`用来记录筛选文件的结果，其中`inputs[2]`记录了参与 compact 的两层文件，是最重要的两个变量

```
  // Each compaction reads inputs from "level_" and "level_+1"
  std::vector<FileMetaData*> inputs_[2];      // The two sets of inputs
```

## 3. `VersionSet::PickCompaction`

`Compaction* VersionSet::PickCompaction()`简言之，就是选取一层需要compact的文件列表，及相关的下层文件列表，记录在`Compaction*`返回。

### 3.1. `size_compaction` or `seek_compaction`

首先是根据`size_compaction seek_compaction`计算应当 compact 的文件。

只有`compaction_score_ >= 1`时，触发 size compaction.

```
  // We prefer compactions triggered by too much data in a level over
  // the compactions triggered by seeks.
  const bool size_compaction = (current_->compaction_score_ >= 1);//文件数过多
  const bool seek_compaction = (current_->file_to_compact_ != nullptr);//seek了多次文件但是没有查到，记录到的file_to_compact_
```

如果`size_compaction = true`，则找到该层一个满足条件的文件：

```
  if (size_compaction) {
    //该层第一个>compact_pointer_的文件，或者第一个文件
    level = current_->compaction_level_;
    assert(level >= 0);
    assert(level+1 < config::kNumLevels);
    c = new Compaction(options_, level);

    // Pick the first file that comes after compact_pointer_[level]
    for (size_t i = 0; i < current_->files_[level].size(); i++) {
      FileMetaData* f = current_->files_[level][i];
      if (compact_pointer_[level].empty() ||
          icmp_.Compare(f->largest.Encode(), compact_pointer_[level]) > 0) {
        c->inputs_[0].push_back(f);
        break;
      }
    }
    if (c->inputs_[0].empty()) {
      // Wrap-around to the beginning of the key space
      c->inputs_[0].push_back(current_->files_[level][0]);
    }

```

`compact_pointer_`是 string 类型，记录了该层上次 compact 时文件的 largest key，初始值为空，也就是选择该层第一个文件。

```
  // Per-level key at which the next compaction at that level should start.
  // Either an empty string, or a valid InternalKey.
  std::string compact_pointer_[config::kNumLevels];
```

如果`seek_compaction = true`，则直接使用满足条件的文件。

到了这一步，`inputs_[0]`里有且仅有一个文件。

### 3.2. level 0 输入文件的特殊处理

level 0 的文件之间是无序的，假设当前有 4 个文件，key range 分别是

1. `[a, n]`  
2. `[c, k]`  
3. `[b, e]`  
4. `[l, n]`  

本次选择了第3个文件，如果只是把`[b, e]`更新到 level 1，那么就会导致读取时数据错误。因为多个文件之间数据是有重叠的，数据之间的先后无法判断，而更新到 level 1 就意味着认为数据更早。

对应的做法就是当选出文件后，判断还有哪些文件有重叠，把这些文件都加入进来，这个例子对应的就是把文件1 2都加进来。

代码上，先通过`GetRange`获取输入文件的 key range，然后根据 key range 得到一个最全的文件列表。

```
  // Files in level 0 may overlap each other, so pick up all overlapping ones
  // 对level 0，获取当前已选择文件的key range: [smallest, largest]
  // 然后选择level 0的其他与该key range有overlap的文件，组成新的key range
  // 然后重新从头在level 0查找，直到key range固定下来
  if (level == 0) {
    InternalKey smallest, largest;
    GetRange(c->inputs_[0], &smallest, &largest);
    // Note that the next call will discard the file we placed in
    // c->inputs_[0] earlier and replace it with an overlapping set
    // which will include the picked file.
    current_->GetOverlappingInputs(0, &smallest, &largest, &c->inputs_[0]);
    assert(!c->inputs_[0].empty());
  }
```

到了这一步，本质上是是的待 compact 的文件在各层都满足统一条件：**`inputs_[0]`的文件跟本层其他文件之间，没有 key 重叠**

### 3.3. SetupOtherInputs

`inputs_[0]`填充了第一层要参与 compact 的文件，接下来就是要计算下一层参与 compact 的文件，记录到`inputs_[1]`。

基本的思想是：所有有重叠的 level + 1 层文件都要参与 compact，得到这些文件后，反过来看下，如果在不增加 level + 1 层文件的前提下，能否增加 level 层的文件？

也就是尽量增加 level 层的文件，贪心算法。

首先是计算下一层与`inputs_[0]` key range 有重叠的所有 sstable files，记录到`inputs_[1]`

```
void VersionSet::SetupOtherInputs(Compaction* c) {
  const int level = c->level();
  InternalKey smallest, largest;
  //inputs_[0]所有文件的key range -> [smallest, largest]
  GetRange(c->inputs_[0], &smallest, &largest);

  //inputs_[1]记录level + 1层所有与inputs_[0]有overlap的文件
  current_->GetOverlappingInputs(level+1, &smallest, &largest, &c->inputs_[1]);

  // Get entire range covered by compaction
  InternalKey all_start, all_limit;
  //inputs_[0, 1]两层所有文件的key range -> [all_start, all_limit]
  GetRange2(c->inputs_[0], c->inputs_[1], &all_start, &all_limit);
```

根据`inputs_[1]`反推下 level 层有多少 key range 有重叠的文件，记录到`expanded0`：

```
  // See if we can grow the number of inputs in "level" without
  // changing the number of "level+1" files we pick up.
  // 如果再不增加level + 1层文件的情况下，尽可能的增加level层的文件
  if (!c->inputs_[1].empty()) {
    std::vector<FileMetaData*> expanded0;
    //level层与[all_start, all_limit]有overlap的所有文件，记录到expanded0
    //expanded0 >= inputs_[0]
    current_->GetOverlappingInputs(level, &all_start, &all_limit, &expanded0);
```

如果文件确实又增加，同时又不会增加太多文件(太多会导致 compact 压力过大)

```
    const int64_t inputs0_size = TotalFileSize(c->inputs_[0]);
    const int64_t inputs1_size = TotalFileSize(c->inputs_[1]);
    const int64_t expanded0_size = TotalFileSize(expanded0);
    //1. level 层参与compact文件数有增加
    //2. 但合并的文件总量在ExpandedCompactionByteSizeLimit之内（防止compact过多）
    if (expanded0.size() > c->inputs_[0].size() &&
        inputs1_size + expanded0_size <
            ExpandedCompactionByteSizeLimit(options_)) {
```

那么就增加参与 compact 的文件，更新到`inputs_`

```
      InternalKey new_start, new_limit;
      //[new_start, new_limit]记录expand0的key range
      GetRange(expanded0, &new_start, &new_limit);
      std::vector<FileMetaData*> expanded1;
      //如果level层文件从inputs_[0]扩展到expand0，key的范围变成[new_start, new_limit]
      //看下level + 1层overlap的文件范围，记录到expand1
      current_->GetOverlappingInputs(level+1, &new_start, &new_limit,
                                     &expanded1);
      //确保level + 1层文件没有增加，那么使用心得expand0, expand1
      if (expanded1.size() == c->inputs_[1].size()) {
        Log(options_->info_log,
            "Expanding@%d %d+%d (%ld+%ld bytes) to %d+%d (%ld+%ld bytes)\n",
            level,
            int(c->inputs_[0].size()),
            int(c->inputs_[1].size()),
            long(inputs0_size), long(inputs1_size),
            int(expanded0.size()),
            int(expanded1.size()),
            long(expanded0_size), long(inputs1_size));
        smallest = new_start;
        largest = new_limit;
        c->inputs_[0] = expanded0;
        c->inputs_[1] = expanded1;
        GetRange2(c->inputs_[0], c->inputs_[1], &all_start, &all_limit);
      }
```

注意这里是个来回判断的过程，我们再梳理一遍：

1. 根据`inputs_[0]`确定`inputs_[1]`  
2. 根据`inputs_[1]`反过来看下能否扩大`inputs_[0]`  
3. `inptus_[0]`扩大的话，记录到`expanded0`  
4. 根据`expanded[0]`看下是否会导致`inputs_[1]`增大  
5. 如果`inputs[1]`没有增大，那就扩大 compact 的 level 层的文件范围  

也就是：

**在不增加 level + 1 层文件，同时不会导致 compact 的文件过大的前提下，尽量增加 level 层的文件数**

到此，参与 compact 的文件集合就已经确定了，为了避免这些文件合并到 level + 1 层后，跟 level + 2 层有重叠的文件太多，届时合并 level + 1 和 level + 2 层压力太大，因此我们还需要记录下 level + 2 层的文件，后续 compact 时用于提前结束的判断：

```
  // Compute the set of grandparent files that overlap this compaction
  // (parent == level+1; grandparent == level+2)
  // level + 2层有overlap的文件，记录到c->grandparents_
  if (level + 2 < config::kNumLevels) {
    //level + 2层overlap的文件记录到c->grandparents_
    current_->GetOverlappingInputs(level + 2, &all_start, &all_limit,
                                   &c->grandparents_);
  }
```

接着记录`compact_pointer_`到`c->edit_`，在后续`PickCompaction`入口时使用。

```
  // Update the place where we will do the next compaction for this level.
  // We update this immediately instead of waiting for the VersionEdit
  // to be applied so that if the compaction fails, we will try a different
  // key range next time.
  // 记录该层本次compact的最大key
  compact_pointer_[level] = largest.Encode().ToString();
  c->edit_.SetCompactPointer(level, largest);
```

最后一步，就是返回筛选的结果`c`:

```
//选取一层需要compact的文件列表，及相关的下层文件列表，记录在Compaction*
Compaction* VersionSet::PickCompaction() {
  Compaction* c;
  ...
  return c;
}
```
