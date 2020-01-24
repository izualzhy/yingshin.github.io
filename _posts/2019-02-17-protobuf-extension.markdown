---
title: "protobuf之extension"
date: 2019-02-17 18:00:06
tags: [protobuf]
---

最近看到的代码里， extension 字段使用越来越广泛，因此这篇笔记介绍下 extension 的使用。同时，[protobuf反射详解](https://izualzhy.cn/protobuf-message-reflection)里的 field 接口获取不到 extension 字段，因此也会介绍下关于 extension 的反射。

## 1. 使用

extension 用于解决字段扩展的场景，例如我们提供一个基础的 message，各个第三方都可以基于该 message 扩展新的字段，例如:

```
// base.proto
package base;
message Link {
  optional string url = 1;
  extensions 1000 to 1050;
}

// third-party-1.proto
package thirdparty1;
extend Link {
  optional int32 weight = 1001;
}

// third-party-2.proto
package thirdparty2;
extend Link {
  optional string ip = 1001;
}
```

使用上主要是4个接口：

```
base::Link link;
link.SetExtension(thirdparty1::weight, 100);
//GetExtension
//HasExtension
//MutableExtension
```

跟原生 field 接口很像，注意这里不是真的构造了一个子类，而是一个 extension 字段的名字，例如：`thirdparty1::weight` `thirdparty2::ip`.

## 2. 反射

与获取普通字段信息的接口`field_count field`不同，extension 字段有单独的接口：

```
    // 基础message定义了多少个extensions range
    int ext_range_count = descriptor->extension_range_count();
    std::cout << "ext_range_count:" << ext_range_count << std::endl;
    // 分别遍历每个range
    for (int i = 0; i < ext_range_count; ++i) {
        // 获取该range的区间
        const google::protobuf::Descriptor::ExtensionRange* ext_range = descriptor->extension_range(i);
        std::cout << "start:" << ext_range->start << " end:" << ext_range->end << std::endl;
        // 遍历该range的区间，获取是否定义了field
        for (int tag_number = ext_range->start; tag_number < ext_range->end; ++tag_number) {
            const google::protobuf::FieldDescriptor* field =
                reflection->FindKnownExtensionByNumber(tag_number);
            if (field != NULL) {
                std::cout << "extend field full_name:" << field->full_name() << std::endl;
            }
        }
    }
```

可以参考 brpc 源码里[PBToJson](https://github.com/brpc/brpc/blob/master/src/json2pb/pb_to_json.cpp)部分，主要是`PbToJsonConverter::Convert`函数，无论是否设置了该字段的值，只要 proto 文件有定义，就能获取到。

不过这里有个比较大的性能隐患，就是`FindKnownExtensionByNumber`，举个例子，如果定义了
```
// base.proto
message BaseData {
    optional int32 id = 1;
    optional bytes url = 2;

    extensions 1000 to 2000;
}

// extension.proto
import "base.proto";

package base_extension;
extend test.BaseData {
    optional string layer = 1000;
}
```

无论是否设置了`base_extension::layer`，尝试调用`ProtoMessageToJson` 1000 次，需要 68ms 左右，而如果普通的`message BaseData`，只需要 1ms。对于一个普通的 message，反射遍历在 us 级别，如果定义了 extensions，例如`extensions 1000 to max`，反射遍历则需要几百秒，性能差别很大。

最近刚好踩过这个坑，花了一个晚上才定位，猜测`FindKnownExtensionByNumber`是个O(n)接口，导致反射的整体实现是O(n2)。定义 proto 时，extension range 需要斟酌下，特别是`extension xxx to max`这种写法。

`ProtoMessageToJson`的场景下，另外一个推荐做法是使用`Reflection::ListFields`接口，不过功能上有 diff，就是只返回了设置了值的字段(`HasField`).

## 3. Tips

### 3.1. 不同 proto 文件 extend 了同一个 tag

如果不同的 proto 文件 extend 了同一个 tag，那么同时编译会有冲突，这种场景下，建议使用 protobuf 的自动编译，运行时加载不同的  proto 文件：

```
#include "google/protobuf/message.h"
#include "google/protobuf/compiler/importer.h"
#include "google/protobuf/dynamic_message.h"

int ProtobufGenerator(
        const google::protobuf::compiler::DiskSourceTree* source_tree,
        google::protobuf::compiler::Importer* importer,
        google::protobuf::DynamicMessageFactory* factory,
        const std::string& proto_file_name,
        const std::string& message_name) {
    const google::protobuf::FileDescriptor* fd = importer->Import(proto_file_name);
    std::cout << "proto_file_name:" << proto_file_name
        << "\n========= DebugString =========\n"
        << fd->DebugString()
        << "==============================="
        << std::endl;

    const google::protobuf::Descriptor* desc = importer->pool()->FindMessageTypeByName(message_name);

    //get message type
    const google::protobuf::Message* message = factory->GetPrototype(desc);
    if (message ==  NULL) {
        fprintf(stderr, "GetPrototype error. proto_file:%s message_name:%s",
                proto_file_name.c_str(),
                message_name.c_str());
        return -1;
    }
    std::cout << "message_name:" << message_name
        << "\nGetTypeName:" << message->GetTypeName()
        << "\nDesc:\n"
        << "============ Desc =============\n"
        << desc->DebugString()
        << "==============================="
        << std::endl;

    //查看具体 field，例如是否存在 url 这个 field
    const google::protobuf::FieldDescriptor* field_desc = message->GetDescriptor()->FindFieldByName("url");
    if (field_desc == NULL) {
        printf("url field not defined.\n");
    } else {
        printf("url field defined.\n");
    }

    return 0;
}

class ErrorCollector : public google::protobuf::compiler::MultiFileErrorCollector {
public:
    virtual void AddError(
            const std::string& filename,
            int line,
            int column,
            const std::string& message) {
        printf("filename:%s line:%d column:%d message:%s",
                filename.c_str(),
                line,
                column,
                message.c_str());
    }
};//ErrorCollector

int main(int argc, char* argv[]) {
    if (argc != 3) {
        fprintf(stderr, "./test_protobuf_generator proto_file_name message_name.");
        return -1;
    }

    const auto source_tree(new google::protobuf::compiler::DiskSourceTree);
    source_tree->MapPath("", ".");
    ErrorCollector error;
    auto importer(new google::protobuf::compiler::Importer(
                source_tree,
                &error));
    auto factory(new google::protobuf::DynamicMessageFactory);

    ProtobufGenerator(
            source_tree,
            importer,
            factory,
            argv[1],
            argv[2]);

    delete factory;
    delete importer;
    delete source_tree;

    return 0;
}
```

2. extend 只是扩展了类

extend 一个原有的类并不会创建一个新的类出来，只是相当于新增字段。例如最开始的例子：

```
extend Link {
  optional int32 weight = 1001;
}
```

```
message AnotherLink {
  optional string url = 1;
  optional weight = 1001;
}
```

`Link` 和 `AnotherLink` 字段设置的值相同的话，序列化的结果就是一样的。
