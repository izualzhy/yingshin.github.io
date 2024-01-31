---
title: 追core笔记之二：如何写出安全的回调
date: 2016-11-23 23:39:16
excerpt: "追core笔记之二：如何写出安全的回调"
tags: core
---

假设我们在模块里使用了sdk来读取远程存储，sdk提供异步读的接口来注册回调、上下文。回调在sdk的线程里调用。

<!--more-->

简化后的模型如下：

```
#include <stdio.h>

namespace tera {
class RowReader;
typedef void (*CallBack)(RowReader* reader);

//RowReader里保存了上下文context，回调函数callback
struct RowReader {
    RowReader(void* context_, CallBack callback_) :
        context(context_),
        callback(callback_) {}
    void* context;
    CallBack callback;
};//RowReader

}//tera

//统一封装了RowReader的操作
class LinkbaseReader {
public:
    LinkbaseReader() {
        printf("LinkbaseReader::LinkbaseReader this:%p\n", this);
    }
    ~LinkbaseReader() {
        printf("LinkbaseReader::~LinkbaseReader this:%p\n", this);
    }

    struct Context {
        Context(LinkbaseReader* linkbase_reader_) :
            linkbase_reader(linkbase_reader_) {}
        LinkbaseReader* linkbase_reader;
    };//Context
    tera::RowReader* read();//简化考虑，返回类型tera::RowReader*
    static void read_callback(tera::RowReader* row_reader);//待注册的回调函数
};//LinkbaseReader

tera::RowReader* LinkbaseReader::read() {
    Context* context = new Context(this);
    return new tera::RowReader(context, &LinkbaseReader::read_callback);//do nothing
}

void LinkbaseReader::read_callback(tera::RowReader* row_reader) {
    LinkbaseReader::Context* context = static_cast<LinkbaseReader::Context*>(row_reader->context);
    printf("LinkbaseReader::read_callback linkbase_reader:%p\n", context->linkbase_reader);
    //linkbase_reader->do_something(); linkbase_reader是否已经delete
    delete row_reader;
    delete context;
}
```

`LinkbaseReader`在模块内部封装了读的操作，对外提供`read`接口读取数据，实现上则构造`RowReader`后设置`context/callback`传给sdk，等待在sdk线程里回调`callback`，其中`context`里设置了`this`来保证在回调里能够继续更新`linkbasereader`。

然而实际场景经常发生这样的问题，由于回调函数`LinkbaseReader::read_callback`是在其他lib的线程里调用，回调时我们的`linkbasereader`可能已经析构，例如：

```
int main() {
    LinkbaseReader* linkbase_reader = new LinkbaseReader();
    tera::RowReader* row_reader = linkbase_reader->read();

    //假装被回调，因为回调在其他线程，delete可能会在回调调用之前
    delete linkbase_reader;
    LinkbaseReader::read_callback(row_reader);//可能使用已经析构的linkbasereader

    return 0;
}
```

程序如我们预想的输出：

```
LinkbaseReader::LinkbaseReader this:0x602010
LinkbaseReader::~LinkbaseReader this:0x602010
LinkbaseReader::read_callback linkbase_reader:0x602010
```

### 1. shared_ptr

我们模块目前的做法是记录当前未返回的回调数目，计数器在`read`里自增，`read_callback`里自减，个人感觉用`shared_ptr`更合适一些。原因是：
使用计数器的方式，我们必须等待计数器清零后才能`delete linkbase_reader`，但有些场景下，我们在程序里并不想一直去做**等待**这种操作。

如果要使用`shared_ptr`，那么在`LinkbaseReader::read`里，怎么获取自身的智能指针呢？这时候就需要神器`enable_shared_from_this`登场了。

修改后的代码如下：

```
#include <stdio.h>
#include <boost/shared_ptr.hpp>
#include <boost/enable_shared_from_this.hpp>

class LinkbaseReader : public boost::enable_shared_from_this<LinkbaseReader> {
public:
    LinkbaseReader() {
        printf("LinkbaseReader::LinkbaseReader this:%p\n", this);
    }
    ~LinkbaseReader() {
        printf("LinkbaseReader::~LinkbaseReader this:%p\n", this);
    }

    struct Context {
        Context(const boost::shared_ptr<LinkbaseReader>& linkbase_reader_) :
            linkbase_reader(linkbase_reader_) {}//传入的是shared_ptr
        boost::shared_ptr<LinkbaseReader> linkbase_reader;
    };//Context
    tera::RowReader* read();
    static void read_callback(tera::RowReader* row_reader);
};//LinkbaseReader

tera::RowReader* LinkbaseReader::read() {
    Context* context = new Context(shared_from_this());
    return new tera::RowReader(context, &LinkbaseReader::read_callback);//do nothing
}

void LinkbaseReader::read_callback(tera::RowReader* row_reader) {
    LinkbaseReader::Context* context = static_cast<LinkbaseReader::Context*>(row_reader->context);
    printf("LinkbaseReader::read_callback linkbase_reader:%p\n", context->linkbase_reader.get());
    //linkbase_reader->do_something();safe，因为context还保留了1个引用计数
    delete context;
    delete row_reader;
}

int main() {
    boost::shared_ptr<LinkbaseReader> linkbase_reader(new LinkbaseReader);
    tera::RowReader* row_reader = linkbase_reader->read();

    //假装被回调
    LinkbaseReader::read_callback(row_reader);

    return 0;//程序退出时linkbase_reader自动删除
}
```

使用`enable_shared_from_this`后，对象必须由`shared_ptr`来管理，好在模块本身的`linkbase_reader`也是一直用智能指针来管理的，这点我觉得非常赞。在大型的项目里，使用raw指针是一件非常危险的事情。不过不要试图用`use_count`来替换掉计数器（如果你的确需要计数器的话），因为`use_count`不提供高效率的操作。

这个版本我们似乎已经解决了问题，但是仔细想下，我们真的需要所有回调返回时才删除`linkbase_reader`回收内存么？

### 2. weak_ptr

修改后版本，我们相当于把`linkbase_reader`的生命周期交给了`RowReader`，如果回调由于某种神秘力量没有调用，那么`linkbase_reader`永远不会析构。或者在某些场景下，我们希望提前`delete linkbase_reader`，回调里做其他处理就好了，这时候需要第二个神器`weak_ptr`登场了。

事实上最开始我觉得在sdk里应该提供这种方式，不过确实很少见sdk提供`shared_ptr`的返回值接口。

```
class LinkbaseReader : public boost::enable_shared_from_this<LinkbaseReader> {
public:
    LinkbaseReader() {
        printf("LinkbaseReader::LinkbaseReader this:%p\n", this);
    }
    ~LinkbaseReader() {
        printf("LinkbaseReader::~LinkbaseReader this:%p\n", this);
    }

    struct Context {
        Context(const boost::shared_ptr<LinkbaseReader>& linkbase_reader_) :
            linkbase_reader(linkbase_reader_) {}
        boost::weak_ptr<LinkbaseReader> linkbase_reader;
    };//Context
    tera::RowReader* read();
    static void read_callback(tera::RowReader* row_reader);
};//LinkbaseReader

tera::RowReader* LinkbaseReader::read() {
    Context* context = new Context(shared_from_this());
    return new tera::RowReader(context, &LinkbaseReader::read_callback);//do nothing
}

void LinkbaseReader::read_callback(tera::RowReader* row_reader) {
    LinkbaseReader::Context* context = static_cast<LinkbaseReader::Context*>(row_reader->context);
    boost::shared_ptr<LinkbaseReader> linkbase_reader = context->linkbase_reader.lock();
    if (linkbase_reader.get() != NULL) {
        printf("LinkbaseReader::read_callback linkbase_reader:%p\n", linkbase_reader.get());
    } else {
        printf("LinkbaseReader::read_callback linkbase_reader has been deleted.\n");
    }
    delete context;
    delete row_reader;
}

int main() {
    tera::RowReader* row_reader = NULL;
    {
        boost::shared_ptr<LinkbaseReader> linkbase_reader(new LinkbaseReader);
        row_reader = linkbase_reader->read();
    }

    //假装被回调，此时linkbase_reader已经析构
    LinkbaseReader::read_callback(row_reader);

    return 0;
}
```

程序输出

```
LinkbaseReader::LinkbaseReader this:0x605010
LinkbaseReader::~LinkbaseReader this:0x605010
LinkbaseReader::read_callback linkbase_reader has been deleted.
```
