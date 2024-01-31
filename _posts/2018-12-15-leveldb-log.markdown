---
title: "leveldb笔记之2:日志"
date: 2018-12-15 16:05:31
excerpt: "leveldb笔记之2:日志"
tags: leveldb
---


我们通常说的日志有两种形式：

1. 记录程序的关键路径日志，用于辅助定位与诊断程序问题，或者简单的统计分析，因此通常是明文的，这类日志又叫做 diagnostic logs, 常用的例如 C++ 里的 [glog](https://izualzhy.cn/glog-source-reading-notes), Java 里的 [Log4j](https://en.wikipedia.org/wiki/Log4j) 等  
2. 记录程序的数据日志，用于进程重启后的数据恢复，或者主备间的数据同步，通常是二进制的，这类日志又叫做 [write-ahead logging](https://en.wikipedia.org/wiki/Write-ahead_logging) ，常见的例如 mysql 的 binlog.  

这篇笔记想要介绍的是第二种日志，也就是[leveldb笔记之基本架构
](https://izualzhy.cn/leveldb-architecture)图里的 log 部分。

## 1. 写日志

前面我们介绍过 leveldb 的几个组件，其中 memtable, immutable memtable 存储在内存。当用户 `Put` 成功后，遇到进程重启、机器重启等问题，如果数据还未来得及 compact 持久化，再 `Get` 时就找不到了。

**这对一个数据库来讲，显然是无法接受的。**

因此，进程重启时必须要能从磁盘（或者其他持久化介质）重新恢复数据，这是数据库实现的常见手段，由于是简单的追加写磁盘，因此写入是非常快的。`Put`操作也不能简单的写入到 memtable，而是要先写日志、再写 memtable，都更新完成后再返回写入成功，write-ahead logging 名字也是由此而来。

leveldb 里的日志写操作主要由`leveldb::log::Writer`完成，因此我们先单独讲讲这个类。

## 2. class log::Writer

`log::Writer`类主要负责：

**接收待写入数据，组织数据格式，调用成员变量完成数据真正写入到文件系统**

构造函数如下：

```
  // Create a writer that will append data to "*dest".
  // "*dest" must be initially empty.
  // "*dest" must remain live while this Writer is in use.
  explicit Writer(WritableFile* dest);
```

我们知道 leveldb 里有大量与文件系统交互的操作，`WritableFile` 定义了顺序写文件操作的虚接口，真正负责写入的是`PosixWritableFile`，该类实现比较简单.

三个类的关系如图

![log::Writer](/assets/images/leveldb/log_writer.png)

`WritableFile PosixWritableFile`封装了`Append/Close/Flush/Sync`的文件操作接口，可以看到写接口上只有`Append`追加写。

`Writer`则提供了`AddRecord`接收字符串，构造写入的数据格式，调用`WritableFile`写入到文件系统。

注：`slice`是对字符串的常用封装，跟之前介绍过的[StringPiece](https://izualzhy.cn/string-piece-introduction)是一套思路。

同时，从这里也可以看到 leveldb 在文件系统上良好的接口设计，支持多平台扩展，例如我们可以实现一个`HdfsWritableFile`，将日志数据更新到 hdfs.

## 3. 源码分析

先介绍下日志的数据格式，如图:

![log::Writer format](/assets/images/leveldb/log_format.png)

可以看到一个完整的 log 由多个 block 组成, block 的大小是固定的:

```
static const int kBlockSize = 32768;//0x8000 = 32k
```

因为日志的主要作用是恢复数据，[log reader](https://github.com/yingshin/leveldb_more_annotation/blob/master/db/log_reader.cc)可以在读取数据时每次固定读取`kBlockSize`，对内存管理及文件读取次数都更友好。

一个 block 内的数据又由多个 {header, data} 组成，其中 data 是用户调用`AddRecord`接口写入的数据，header 占用7个字节，分别记录了: data 的数据签名、数据长度及 type.

```
// Header is checksum (4 bytes), length (2 bytes), type (1 byte).
// 其中length最大为kBlockSize=0x8000 - kHeaderSize，因此只使用2个字节存储
static const int kHeaderSize = 4 + 2 + 1;
```

由于单个 block 大小有限制，因此写入数据时，可能无法完全写入当前 block.解决方案是多次写入，使用不同的 type 来标记是第一次/中间/最后一次写入。

例如当前 block 只有10个字节，而用户调用了`AddRecord('HelloWorld')`，数据会分为两次写入，type 分别为 kFirstType kLastType:

```
**************  block N  **************
...
|crc 3 kFirstType|hel|
************** block N+1 **************
|crc 7 kLastType|loWorld|
...
```

可以看到先尝试写入`hel`填满该 block，标记这次写入为`kFirstType`，总共占用 10 个字节。然后写入`loWorld`到下一个 block，标记为`kLastType`.

如果这个 block 写不下，那么就标记为`kMiddleType`.

type 是一个枚举值：

```
enum RecordType {
  // Zero is reserved for preallocated files
  kZeroType = 0,

  kFullType = 1,

  // For fragments
  kFirstType = 2,
  kMiddleType = 3,
  kLastType = 4
};
```

如果 block 剩余的空间不足7 bytes，写不下 header，那么就补`\0`填满。

上述过程代码添加了注释，位置在 [log_writer.cc](https://github.com/yingshin/leveldb_more_annotation/blob/master/db/log_writer.cc).


读日志由[log::Reader](https://github.com/yingshin/leveldb_more_annotation/blob/master/db/log_reader.cc)完成，与`Writer`过程正好相反。

## 4. 应用示例

主要用两个例子来说明。

[log_writer_test](https://github.com/yingshin/leveldb_more_annotation/blob/master/my_test/log_writer_test.cpp) 添加一段 data 写入到文件，然后分析文件里每个字节的含义。

```
#include <iostream>
#include "db/log_writer.h"
#include "leveldb/env.h"
#include "util/crc32c.h"
#include "util/coding.h"

int main() {
    std::string file_name("log_writer.data");

    leveldb::WritableFile* file;
    leveldb::Status s = leveldb::Env::Default()->NewWritableFile(
            file_name,
            &file);

    leveldb::log::Writer writer(file);

    const std::string data = "HelloWorld";
    s = writer.AddRecord(data);//字符串长度10=0x0a
    std::cout << s.ToString() << std::endl;

    delete file;

    return 0;
}
```

xxd看下写入的文件

```
$ xxd log_writer.data
00000000: 0a06 1c77 0a00 0148 656c 6c6f 576f 726c  ...w...HelloWorl
00000010: 64                                       d
```

1. checksum: 最开始4个 bytes `0a06 1c77`
2. 数据长度：`0a00`，占2个bytes
3. type: `01`，即`kFullType`
4. 接下来就是字符串了

[log_writer_blob_test](https://github.com/yingshin/leveldb_more_annotation/blob/master/my_test/log_writer_blob_test.cpp) 用来看下多种 type 的情况:

```
#include <iostream>
#include "db/log_writer.h"
#include "leveldb/env.h"
#include "util/crc32c.h"
#include "util/coding.h"

int main() {
    std::string file_name("log_writer_blob.data");

    leveldb::WritableFile* file;
    leveldb::Status s = leveldb::Env::Default()->NewWritableFile(
            file_name,
            &file);

    leveldb::log::Writer writer(file);

    std::string data(leveldb::log::kBlockSize - leveldb::log::kHeaderSize - 10, 'a');
    s = writer.AddRecord(data);//字符串长度32751 = 0x7fef
    std::cout << s.ToString() << std::endl;

    data.assign("HelloWorld");
    s = writer.AddRecord(data);

    delete file;

    return 0;
}
```

xxd看下写入文件的最后几行

```
$ xxd log_writer_blob.data | tail
00007f70: 6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
00007f80: 6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
00007f90: 6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
00007fa0: 6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
00007fb0: 6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
00007fc0: 6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
00007fd0: 6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
00007fe0: 6161 6161 6161 6161 6161 6161 6161 6161  aaaaaaaaaaaaaaaa
00007ff0: 6161 6161 6161 e2dc 0e4f 0300 0248 656c  aaaaaa...O...Hel
00008000: 8ef2 3671 0700 046c 6f57 6f72 6c64       ..6q...loWorld
```

1. checksum: 一串 a 之后的4个 bytes `e2dc 0e4f`
2. 数据长度：`0300`，占2个bytes
3. type: `02`，即`kFirstType`
4. 字符串: `Hel`
5. 在下一个 block 写入字符串`loWorld`及 header.

## 5. 在 leveldb 中的位置

`log::Writer`作为一个单独的类已经介绍完了，在 level 中，[db_impl.cc](https://github.com/yingshin/leveldb_more_annotation/blob/master/db/db_impl.cc)会构造`log::Writer`对象并且写入数据:

```
    log_ = new log::Writer(lfile);

    ...

      //WriterBatch写入log文件，包括:sequence,操作count,每次操作的类型(Put/Delete)，key/value及其长度
      status = log_->AddRecord(WriteBatchInternal::Contents(updates));
      bool sync_error = false;
      if (status.ok() && options.sync) {
        //log_底层使用logfile_与文件系统交互，调用Sync完成写入
        status = logfile_->Sync();
        if (!status.ok()) {
          sync_error = true;
        }
      }
```

`WriteBatchInternal::Contents(updates)`数据格式图里的 data 部分，这些究竟包含了什么？可以参考[写入与读取流程](https://izualzhy.cn/leveldb-write-read#3-writebatch)这篇笔记。

## 6. 参考资料

1. [leveldb Log format](https://github.com/yingshin/leveldb_more_annotation/blob/master/doc/log_format.md)
2. [leveldb 日志](https://leveldb-handbook.readthedocs.io/zh/latest/journal.html)
