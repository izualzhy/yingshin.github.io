---
title: "leveldb笔记之6:block读取"
date: 2018-12-16 21:53:59
excerpt: "leveldb笔记之6:block读取"
tags: leveldb
---

## 1. 简介

[BlockBuilder](https://izualzhy.cn/leveldb-block)写入后的数据，是通过[class Block](https://github.com/yingshin/leveldb_more_annotation/blob/master/table/block.cc)解析完成的。本文通过介绍读取的过程，能够帮助读者更深入的理解一个 Block 的数据格式。

## 2. 再谈 restart point

构造 Block时，使用 share key 可以有效的减少存储需要的字节数，但是对应的，解析一个 key 时需要从第一个 key 一直计算过来，如果中间的 key 很多，那么显然是无法接受的，因此可以说 share key 对存储是友好的，但是不利于读取。

所以，引入 restart 的一个作用就是控制中间 key 的数量。

一个 block 的数据格式如图所示：

![BlockBuilderLogic](/assets/images/leveldb/block_builder_logic.png)

可以看到，所有的 key(或者说entry) 通过 restart points 分成了若干区间，每个区间有相同数目的 entry，这个数目即`Options.block_restart_interval`。

当查找 key 时，最朴素的做法即逐个读取，直到遇到符合条件的 key.

但是借助 block 内 key 是有序的这个特性，通过对比 restart point 所指向的 key，我们可以用二分法快速的定位：

**要查找的 target 可能位于哪个 restart point 所指向的区间**

由于 `BlockBuilder` 输出的数据里，最后4 bytes 记录了 restart points 的个数N，再往前 4*N bytes 记录了 restart points，因此我们可以很容易的读取到 restart points.

## 3. 源码解析

`class Block`非常简洁：

```
class Block {
 public:
  // Initialize the block with the specified contents.
  explicit Block(const BlockContents& contents);

  ~Block();

  size_t size() const { return size_; }//contents.data.size
  Iterator* NewIterator(const Comparator* comparator);
  ...
```

构造函数用于传入待解析的数据，即`BlockBuilder::Finish`返回的字符串。

leveldb 里大量使用了 Iterator，Block 的读操作实际上交给 `leveldb::Block::Iter` 完成的，通过`NewIterator`构造，使用后需要手动 delete.

`Iter`定义在[block.cc](https://github.com/yingshin/leveldb_more_annotation/blob/master/table/block.cc)，通过 Iter 的接口就可以实现读取 Block 的各种操作。

### 3.1. DecodeEntry

在介绍`Iter`之前，先介绍下解析单条 entry 的函数

```
static inline const char* DecodeEntry(const char* p, const char* limit,
                                      uint32_t* shared,
                                      uint32_t* non_shared,
                                      uint32_t* value_length) {
  if (limit - p < 3) return nullptr;
  //先假定三个长度都只用1bytes存储，尝试下快速解析
  *shared = reinterpret_cast<const unsigned char*>(p)[0];
  *non_shared = reinterpret_cast<const unsigned char*>(p)[1];
  *value_length = reinterpret_cast<const unsigned char*>(p)[2];
  if ((*shared | *non_shared | *value_length) < 128) {
    // Fast path: all three values are encoded in one byte each
    p += 3;
  } else {
    //逐个解析3个长度
    if ((p = GetVarint32Ptr(p, limit, shared)) == nullptr) return nullptr;
    if ((p = GetVarint32Ptr(p, limit, non_shared)) == nullptr) return nullptr;
    if ((p = GetVarint32Ptr(p, limit, value_length)) == nullptr) return nullptr;
  }

  if (static_cast<uint32_t>(limit - p) < (*non_shared + *value_length)) {
    return nullptr;
  }
  //p此时指向non-shared key
  return p;
}
```

按照`|shared len|non_shared len|value len|non_shared key|value|`解析传入的buffer，返回指向`non_shared key`的指针。

### 3.2. Seek

`Seek`应该是`Block::Iter`最重要的一个接口，用于查找第一个 >= target的 entry.

首先是二分查找刚好小于 target 的 restart point.

```
    // Binary search in restart array to find the last restart point
    // with a key < target
    uint32_t left = 0;
    uint32_t right = num_restarts_ - 1;
    //查找刚好 < target的restart point.
    while (left < right) {
      uint32_t mid = (left + right + 1) / 2;
      uint32_t region_offset = GetRestartPoint(mid);
      uint32_t shared, non_shared, value_length;
      //region_offset是一个restart point，因此:
      //shared = 0, non_shared就是key的长度, key_ptr指向的就是完整的key
      const char* key_ptr = DecodeEntry(data_ + region_offset,
                                        data_ + restarts_,
                                        &shared, &non_shared, &value_length);
      if (key_ptr == nullptr || (shared != 0)) {
        CorruptionError();
        return;
      }
      Slice mid_key(key_ptr, non_shared);
      if (Compare(mid_key, target) < 0) {
        // Key at "mid" is smaller than "target".  Therefore all
        // blocks before "mid" are uninteresting.
        left = mid;
      } else {
        // Key at "mid" is >= "target".  Therefore all blocks at or
        // after "mid" are uninteresting.
        right = mid - 1;
      }
    }
```

注：二分有疑问的话，可以参考[leveldb 二分法](https://izualzhy.cn/leveldb-binary-search)

此时`left`满足条件：

left 指向的 key 一定 < target, left + 1(如果存在的话)指向的key一定 >= target.

接着调用`SeekToRestartPoint(left)`记录`restart_index_`，这个函数主要是配合`ParseNextKey`使用，直到找到第一个 >= target 的 entry 返回，或者达到 block 末尾。

```
    // left指向的key一定<target, left + 1(如果存在)指向的key一定>=target.
    // Linear search (within restart block) for first key >= target
    SeekToRestartPoint(left);
    while (true) {
      if (!ParseNextKey()) {
        return;
      }
      //找到第一个>= target的entry
      if (Compare(key_, target) >= 0) {
        return;
      }
    }
```

### 3.3. ParseNextKey

`ParseNextKey`就是把 iter 指向下一个 entry.

首先获取下一个 entry 的 offset

```
  bool ParseNextKey() {
    current_ = NextEntryOffset();//如果刚调用过SeekToRestartPoint，此时返回第一条entry
    //解析current_指向的entry
    const char* p = data_ + current_;
    const char* limit = data_ + restarts_;  // Restarts come right after data
    if (p >= limit) {
      // No more entries to return.  Mark as invalid.
      current_ = restarts_;
      restart_index_ = num_restarts_;
      return false;
    }
```

然后解析出`key_ value_`

```
    // Decode next entry
    uint32_t shared, non_shared, value_length;
    p = DecodeEntry(p, limit, &shared, &non_shared, &value_length);
    if (p == nullptr || key_.size() < shared) {
      CorruptionError();
      return false;
    } else {
      //提取key value
      key_.resize(shared);
      key_.append(p, non_shared);
      value_ = Slice(p + non_shared, value_length);
```

然后更新`restart_index_`

```
      //调用NextEntryOffset后，current_可能已经指向了下一个restart point区间的内容
      //更新restart_index_，这里用if就可以吧？
      //<是不是应该换成<=?否则只有到了第二条entry才会修正restart_index_
      while (restart_index_ + 1 < num_restarts_ &&
             GetRestartPoint(restart_index_ + 1) < current_) {
        ++restart_index_;
      }
      return true;
    }
  }
};
```

注：感觉这里`restart_index_`不对，不过倒是没有产生 bug.

### 3.4. SeekToFirst

seek 到第一个 restart point，解析对应的 entry.

```
  virtual void SeekToFirst() {
    SeekToRestartPoint(0);
    ParseNextKey();
  }
```

###  3.5. SeekToLast

seek 到最后一个 restart point，解析最后一个 entry.

```
virtual void SeekToLast() {
    SeekToRestartPoint(num_restarts_ - 1);
    while (ParseNextKey() && NextEntryOffset() < restarts_) {
      // Keep skipping
    }
  }
```

### 3.6. Prev

```
  virtual void Prev() {
    assert(Valid());

    // Scan backwards to a restart point before current_
    const uint32_t original = current_;
    // prev entry所在的restart_index_一定< original
    // 不过什么操作会导致while condition为true？
    while (GetRestartPoint(restart_index_) >= original) {
      if (restart_index_ == 0) {
        // No more entries
        current_ = restarts_;
        restart_index_ = num_restarts_;
        return;
      }
      restart_index_--;
    }

    //restart_index_指向current_所在的 or 上一个restart point区间
    SeekToRestartPoint(restart_index_);
    do {
      // Loop until end of current entry hits the start of original entry
    } while (ParseNextKey() && NextEntryOffset() < original);
  }
```

## 4. 例子

参考 [block_builder_test](https://github.com/yingshin/leveldb_more_annotation/blob/master/my_test/block_builder_test.cpp)`test_block`部分，主要就是通过前面介绍的`Iter`各个接口读取`BlockBuilder`写入的数据。
