---
title: "leveldb笔记之12:LRUCache的使用"
date: 2019-02-16 18:31:54
tags: [leveldb]
---

[上篇](https://izualzhy.cn/leveldb-cache)讲了 leveldb 里 LRUCache 的实现，这篇笔记继续介绍下具体的使用场景。

我最开始的理解，是 leveldb 缓存了用户写入/读取的原始 key:value 数据，实际上是错误的。leveldb 缓存了两种类型的数据：[Block](https://github.com/yingshin/leveldb_more_annotation/blob/master/table/block.h)及[Table](https://github.com/yingshin/leveldb_more_annotation/blob/master/include/leveldb/table.h).

接下来分别介绍下.


## 1. Block

通过内存缓存高频访问的 data block，避免从磁盘读取，从而提高数据的访问速率。

### 1.1. 初始化

在`SanitizeOptions`函数里初始化

```
  if (result.block_cache == nullptr) {
    // 默认缓存大小为8M
    result.block_cache = NewLRUCache(8 << 20);
  }
```

总大小默认 8M，单个 block 按照`block_size(4096),`默认大小，一共大概能存储 2048 个 block.

### 1.2. 使用

一个[block](https://izualzhy.cn/leveldb-block)的读取，通过[class Block](https://izualzhy.cn/leveldb-block-read)实现，而[BlockReader](https://izualzhy.cn/leveldb-table#322-tableblockreader)则会构造初始化具体的`Block`对象。

当需要读取一个 block 时，查看是否在 cache 中，如果在，则直接从 cache 中返回`Block*`，否则构造一个`Block*`插入到 cache 并返回。

当我们初始化`Table`时，会从`LRUCache`获取一个全局唯一的 ID (自增int):

```
rep->cache_id = (options.block_cache ? options.block_cache->NewId() : 0);//获取一个全局唯一的ID
```

缓存 key 使用 cache_id + offset 来唯一表示一个 Block

```
      // cache key = (cache_id + offset)
      // cache_id不同Table间保证唯一
      // 同一Table的不同data block有唯一的offset
      // 因此可以作为cache key.
      char cache_key_buffer[16];
      EncodeFixed64(cache_key_buffer, table->rep_->cache_id);
      EncodeFixed64(cache_key_buffer+8, handle.offset());
      Slice key(cache_key_buffer, sizeof(cache_key_buffer));
```

缓存 value 则是整个 Block 对象

```
        s = ReadBlock(table->rep_->file, options, handle, &contents);
        if (s.ok()) {
          block = new Block(contents);
          if (contents.cachable && options.fill_cache) {//本次DB::Get结果是否充缓存
            cache_handle = block_cache->Insert(
                key, block, block->size(), &DeleteCachedBlock);
          }
        }
```

注意`ReadOptions`会配置本次读取是否充缓存。

`Insert`的`charge`参数是整个 block 的大小，因此能够限制内存使用量。  
`BlockReader`返回的是一个迭代器，随着迭代器的销毁，`cache::Handle`会 Release:

```
iter->RegisterCleanup(&ReleaseBlock, block_cache, cache_handle);
```

而随着数据的淘汰，会调用传入的`deleter`函数销毁`Block*`:

```
static void DeleteCachedBlock(const Slice& key, void* value) {
  Block* block = reinterpret_cast<Block*>(value);
  delete block;
}
```

## 2. Table

通过内存缓存高频打开的文件句柄及部分内容，避免从磁盘读取，从而提高 sstable 文件的访问速率，同时控制打开的文件句柄数目。

### 2.1. 初始化

在`DBImpl`初始化列表里构造

```
table_cache_(new TableCache(dbname_, options_, TableCacheSize(options_))),
```

默认大小为`1000 - 10`

```
static int TableCacheSize(const Options& sanitized_options) {
  // Reserve ten files or so for other uses and give the rest to TableCache.
  // max_open_files 默认1000
  return sanitized_options.max_open_files - kNumNonTableCacheFiles;
}
```

### 2.2. 使用

一个[sstable](https://izualzhy.cn/leveldb-sstable)的读取，通过[class Table](https://izualzhy.cn/leveldb-table)实现。


而[Open一个Table](https://izualzhy.cn/leveldb-table#31-open)，主要就是传入文件返回`Table*`，因此缓存的 key 就是文件的 file_number

```
  char buf[sizeof(file_number)];
  EncodeFixed64(buf, file_number);
  Slice key(buf, sizeof(buf));
```

value 就是文件和`Table*`

```
struct TableAndFile {
  RandomAccessFile* file;
  Table* table;
};
```

当需要读取一个 sstable 时，查看是否在 cache 中，如果在，则直接从 cache 中返回`TableAndFile*`，否则构造一个`TableAndFile*`插入到 cache 并返回。

```
      TableAndFile* tf = new TableAndFile;
      tf->file = file;
      tf->table = table;
      *handle = cache_->Insert(key, tf, 1, &DeleteEntry);
```

当数据过期淘汰时，关闭文件句柄，清理内存。

```
static void DeleteEntry(const Slice& key, void* value) {
  TableAndFile* tf = reinterpret_cast<TableAndFile*>(value);
  delete tf->table;
  delete tf->file;
  delete tf;
}
```

`Insert`时`charge`为1，代表一个文件，总的数目大小在初始化时通过`TableCacheSize(options_)`指定，因此就可以起到控制整个进程打开的文件句柄数目的作用。

注意`TableCache`有手动逐出`Evict`的操作，对应删除文件后删除对应缓存的场景。

```
void TableCache::Evict(uint64_t file_number) {
  char buf[sizeof(file_number)];
  EncodeFixed64(buf, file_number);
  cache_->Erase(Slice(buf, sizeof(buf)));
}
```

## 3. 总结

可以看到 leveldb 里是二级缓存，第一级存放`TableAndFile`，第二级存放`Block`，默认都使用 LRUCache，当然也可以自定义。
