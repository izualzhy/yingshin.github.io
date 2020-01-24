---
title: "leveldb笔记之4:MemTable"
date: 2018-12-15 21:22:24
excerpt: "leveldb笔记之4:MemTable"
tags: [leveldb]
---

前面介绍了[ skiplist ](https://izualzhy.cn/skiplist-leveldb)，skiplist 是 leveldb 里一个非常重要的数据结构，实现了高效的数据查找与插入(O(logn))。

但是 skiplist 的最小元素是`Node`，存储单个元素。而 leveldb 是一个单机的 {key, value} 数据库，那么 kv 数据如何作为`Node`存储到 skiplist？如何比较数据来实现查找的功能？leveldb 的各个接口及参数，是如何生效到 skiplist的？

这些问题，在`MemTable`中能够找到答案。

## 1. MemTable 简述

使用 leveldb 写入的数据，为了提供更快的查找性能，部分数据在内存中也会存储一份，这部分数据就使用`MemTable`存储。

`MemTable`初始化时接收一个用于比较的对象，该对象可以比较 key 用于 skiplist 的排序。

```cpp
explicit MemTable(const InternalKeyComparator& comparator);
```

与 leveldb 对外的接口类似，`MemTable`也提供了`Add` `Get`两个接口

```cpp
void Add(SequenceNumber seq, ValueType type,
        const Slice& key,
        const Slice& value);

bool Get(const LookupKey& key, std::string* value, Status* s);
```

其中`key` `value`即用户指定的值，`seq`是一个整数值，随着写入操作递增，因此`seq`越大，该操作越新。`type`表示操作类型：

1. 写入
2. 删除。

`Status`是 leveldb 的一个辅助类，在返回值的表示上更加丰富。类本身比较简单，不过设计还是挺巧妙的，之后会写篇笔记介绍下。

同时`MemTable`还通过迭代器的形式，暴露底层的 skiplist :

```cpp
Iterator* MemTable::NewIterator() {
  return new MemTableIterator(&table_);
}
```

因此`MemTableIterator`实现上也是操作 skiplist 的迭代器。

`table_`就是底层的[ skiplist ](https://izualzhy.cn/skiplist-leveldb)，负责管理数据。

```cpp
  struct KeyComparator {
    const InternalKeyComparator comparator;
    explicit KeyComparator(const InternalKeyComparator& c) : comparator(c) { }
    int operator()(const char* a, const char* b) const;
  };
  friend class MemTableIterator;
  friend class MemTableBackwardIterator;

  typedef SkipList<const char*, KeyComparator> Table;

  KeyComparator comparator_;
  int refs_;
  Arena arena_;
  Table table_;
```

存储类型为`const char*`，比较方式为`KeyComparator`。

接下来分别介绍下。

## 2. 存储的数据格式

用户调用 leveldb 接口指定的 key，到了`MemTable`增加了`SequenceNum`和`ValueType`，以及指定的 value，这些数据都会存储到 skiplist，具体格式如下：

![memtable key](assets/images/memtable_key.png)

varint encode的操作在[protobuf varint](https://izualzhy.cn/protobuf-encode-varint-and-zigzag) 里也介绍过，主要的特点是：

1. 整数由定长改为变长存储
2. 小整数仅占用1个字节，随着数值变大占用的字节数也变大，最多占用5个字节。

具体原理可以参考上面的笔记。

存储数据的整体思路跟 protobuf 序列化类似，还是`|len(key)  |key  |len(value)  |value  |`的格式，然后是`encode` `sequence` `type`这些技巧。

`sequence`和`type`分别占用7bytes和1bytes，其中`sequence`是一个随着写入操作单调递增的值：

```cpp
typedef uint64_t SequenceNumber;
```

`type`是一个枚举变量，表示写入/删除操作

```cpp
// Value types encoded as the last component of internal keys.
// DO NOT CHANGE THESE ENUM VALUES: they are embedded in the on-disk
// data structures.
enum ValueType {
  kTypeDeletion = 0x0,
  kTypeValue = 0x1
};

//用于查找
static const ValueType kValueTypeForSeek = kTypeValue;
```

如注释所说，`type`作为 internal key 的最后一个 component

`MemTable`相关的有多种 key，本质上就是上图里三种：

1. userkey: 用户传入的 key，即用户key.
2. internal key: 增加了`sequence` `type`，用于支持同一 key 的多次操作，以及 snapshot，即内部key.
3. memtable key: 增加了 varint32 encode 后的 internal key长度

用于辅助的类有多个，例如`LookupKey` `ParsedInternalKey` `InternalKey`，其中跟`MemTable`用到了`LookupKey`

```cpp
class LookupKey {
 public:
  // Initialize *this for looking up user_key at a snapshot with
  // the specified sequence number.
  LookupKey(const Slice& user_key, SequenceNumber sequence);

  ~LookupKey();

  // Return a key suitable for lookup in a MemTable.
  Slice memtable_key() const { return Slice(start_, end_ - start_); }

  // Return an internal key (suitable for passing to an internal iterator)
  Slice internal_key() const { return Slice(kstart_, end_ - kstart_); }

  // Return the user key
  Slice user_key() const { return Slice(kstart_, end_ - kstart_ - 8); }

 private:
  // We construct a char array of the form:
  //    klength  varint32               <-- start_
  //    userkey  char[klength]          <-- kstart_
  //    tag      uint64 //sequence && ValueType合称tag
  //                                    <-- end_
  // The array is a suitable MemTable key.
  // The suffix starting with "userkey" can be used as an InternalKey.
  const char* start_;
  const char* kstart_;
  const char* end_;
  //为了避免内存碎片 or 提高性能？
  char space_[200];      // Avoid allocation for short keys
    };
```

对照前面的图 或者 注释，可以找到几个变量`start_ kstart_ end_`的位置。

对比下构造函数看下代码更清楚些(shou me the code.🙂)

```
static uint64_t PackSequenceAndType(uint64_t seq, ValueType t) {
  assert(seq <= kMaxSequenceNumber);
  assert(t <= kValueTypeForSeek);
  return (seq << 8) | t;
}

LookupKey::LookupKey(const Slice& user_key, SequenceNumber s) {
  size_t usize = user_key.size();
  //SequenceNumber + ValueType占8个字节，encode(internal_key_size)至多占5个字节，因此预估至多+13个字节
  size_t needed = usize + 13;  // A conservative estimate
  char* dst;
  if (needed <= sizeof(space_)) {
    dst = space_;
  } else {
    dst = new char[needed];
  }
  //start_指向最开始位置
  start_ = dst;
  //memtable的internal_key_size=usize+8，对应MemTable::Add实现。
  dst = EncodeVarint32(dst, usize + 8);
  //kstart_指向userkey开始位置
  kstart_ = dst;
  memcpy(dst, user_key.data(), usize);
  dst += usize;
  EncodeFixed64(dst, PackSequenceAndType(s, kValueTypeForSeek));
  dst += 8;
  //end_指向(s << 8 | type)的下一个字节
  end_ = dst;
}
```

`EncodeFixed64`如果是小端，则直接内存 copy；如果是大端，则反向 copy，固定占用8bytes。

## 3. 数据怎么比较

整条数据写入 skiplist 后，使用 naive 的比较方式肯定行不通：

1. 前面字节是 internal key经过 varint encode 后的长度，比较这个长度没有意义。
2. 用户如果写入两次，例如先 add {"JeffDean": "Google"}，然后 add {"Jeff Dean"："Bidu"}，预期查找的结果肯定是后者。

在`MemTable`里，使用`KeyComparator`比较，作为`Skiplist`的第二个模板参数

```cpp
  struct KeyComparator {
    const InternalKeyComparator comparator;
    explicit KeyComparator(const InternalKeyComparator& c) : comparator(c) { }
    int operator()(const char* a, const char* b) const;
  };
```

`operator()`负责解析出 internal key，交给 comparator 比较:

```cpp
int MemTable::KeyComparator::operator()(const char* aptr, const char* bptr)
    const {
  // Internal keys are encoded as length-prefixed strings.
  // 分别获取internal_key并比较
  Slice a = GetLengthPrefixedSlice(aptr);
  Slice b = GetLengthPrefixedSlice(bptr);
  return comparator.Compare(a, b);
}
```

可以看到调用了`InternalKeyComparator::Compare`接口：

```cpp
class InternalKeyComparator : public Comparator {
 private:
  const Comparator* user_comparator_;
 public:
  explicit InternalKeyComparator(const Comparator* c) : user_comparator_(c) { }
  virtual const char* Name() const;
  //从slice里解析出userkey，与8字节的tag:(sequence << 8) | type
  //userkey按照字母序比较
  //如果userkey相同，则tag按大小逆序比较，即sequence越大越靠前
  virtual int Compare(const Slice& a, const Slice& b) const;
  ...
};
```

其中`user_comparator_`默认为`BytewiseComparator`，即按照字节大小排序。

`Compare`具体的实现：

```
int InternalKeyComparator::Compare(const Slice& akey, const Slice& bkey) const {
  // Order by:
  //    increasing user key (according to user-supplied comparator)
  //    decreasing sequence number
  //    decreasing type (though sequence# should be enough to disambiguate)
  //    首先比较userkey，即用户写入的key
  int r = user_comparator_->Compare(ExtractUserKey(akey), ExtractUserKey(bkey));
  // 如果是同一个userkey，继续判断tag是否相等
  if (r == 0) {
    //sequence越大，排序结果越小
    const uint64_t anum = DecodeFixed64(akey.data() + akey.size() - 8);
    const uint64_t bnum = DecodeFixed64(bkey.data() + bkey.size() - 8);
    if (anum > bnum) {
      r = -1;
    } else if (anum < bnum) {
      r = +1;
    }
  }
  return r;
}
```

总结一下主要是两条：

1. userkey 按照指定的排序方式，默认字节大小排序，akey-userkey < bkey-userkey 则返回-1.
2. 如果 userkey 相等，那么解析出 sequence，按照 sequence 大小逆序排序，即 akey-sequence > bkey-sequence 则返回-1.sequence越大则代表数据越新，这样的好处是越新的数据越排在前面。

注：`DecodeFixed64`解析出的实际上是`sequence << 8 | type`，因为 `sequence` 占高位，因此直接比较大小也能代表`sequence`。

## 4. Add/Get操作

`Add`过程的代码就是组装memtable key，然后调用`SkipList`接口写入。

```
void MemTable::Add(SequenceNumber s, ValueType type,
                   const Slice& key,
                   const Slice& value) {
  // Format of an entry is concatenation of:
  //  key_size     : varint32 of internal_key.size()
  //  key bytes    : char[internal_key.size()]
  //  value_size   : varint32 of value.size()
  //  value bytes  : char[value.size()]
  size_t key_size = key.size();
  size_t val_size = value.size();
  //InternalKey长度 = UserKey长度 + 8bytes(存储SequenceNumber + ValueType)
  size_t internal_key_size = key_size + 8;
  //用户写入的key value在内部实现里增加了sequence type
  //而到了MemTable实际按照以下格式组成一段buffer
  //|encode(internal_key_size)  |key  |sequence  type  |encode(value_size)  |value  |
  //这里先计算大小，申请对应大小的buffer
  const size_t encoded_len =
      VarintLength(internal_key_size) + internal_key_size +
      VarintLength(val_size) + val_size;
  char* buf = arena_.Allocate(encoded_len);
  //append key_size
  char* p = EncodeVarint32(buf, internal_key_size);
  //append key bytes
  memcpy(p, key.data(), key_size);
  p += key_size;
  //append sequence_number && type
  //64bits = 8bytes，前7个bytes存储s，最后一个bytes存储type.
  //这里8bytes对应前面 internal_key_size = key_size + 8
  //也是Fixed64而不是Varint64的原因
  EncodeFixed64(p, (s << 8) | type);
  p += 8;
  //append value_size
  p = EncodeVarint32(p, val_size);
  //append value bytes
  memcpy(p, value.data(), val_size);
  assert(p + val_size == buf + encoded_len);
  //写入table_的buffer包含了key/value及附属信息
  table_.Insert(buf);
}
```

`Get`通过`SkipList::Iterator::Seek`接口获取第一个`operator >=`的 Node。传入的第一个参数是`LookupKey`包含了 userkey，同时指定了一个较大的`SequenceNumber s`（具体多么大我们后续分解），而根据`InternalKeyComparator`的定义，返回值有两种情况：

1. 如果该 userkey 存在，返回小于 s 的最大 sequence number 的 Node.
2. 如果 userkey 不存在，返回第一个 > userkey 的 Node.

```
bool MemTable::Get(const LookupKey& key, std::string* value, Status* s) {
  Slice memkey = key.memtable_key();
  Table::Iterator iter(&table_);
  iter.Seek(memkey.data());
  if (iter.Valid()) {
    // entry format is:
    //    klength  varint32
    //    userkey  char[klength]
    //    tag      uint64
    //    vlength  varint32
    //    value    char[vlength]
    // Check that it belongs to same user key.  We do not check the
    // sequence number since the Seek() call above should have skipped
    // all entries with overly large sequence numbers.
    const char* entry = iter.key();
    uint32_t key_length;
    //解析出internal_key的长度存储到key_length
    //key_ptr指向internal_key
    const char* key_ptr = GetVarint32Ptr(entry, entry+5, &key_length);
    //Seek返回的是第一个>=key的Node(>= <=> InternalKeyComparator::Compare)
    //因此先判断下userkey是否相等
    if (comparator_.comparator.user_comparator()->Compare(
            Slice(key_ptr, key_length - 8),
            key.user_key()) == 0) {
      // Correct user key
      // tag = (s << 8) | type
      const uint64_t tag = DecodeFixed64(key_ptr + key_length - 8);
      //type存储在最后一个字节
      switch (static_cast<ValueType>(tag & 0xff)) {
        case kTypeValue: {
          Slice v = GetLengthPrefixedSlice(key_ptr + key_length);
          value->assign(v.data(), v.size());
          return true;
        }
        case kTypeDeletion:
          *s = Status::NotFound(Slice());
          return true;
      }
    }
  }
  return false;
}
```

## 5. 参考

本文`MemTable`的注释都ci到了[leveldb_more_annotation](https://github.com/yingshin/leveldb_more_annotation/blob/master/db/memtable.cc)
