---
title: "protobuf 之 Custom Options"
date: 2019-06-02 21:03:42
tags: [protobuf]
---

## 1. 问题

protobuf 默认支持的数据类型有 double float int32 int64 string bytes bool [等几种类型](https://developers.google.com/protocol-buffers/docs/proto#scalar)。在有的生产场景中，我们可能需要更多的类型，比如把 protobuf 转换为 mcpack(厂内某个很古老的数据格式)，对于一段字符串，protobuf 统一认为是 string/bytes，而 mcpack 则把字符串区分为 raw/string 两种类型，此时就需要我们在 proto 能够标记以区分这两种类型。在 [brpc 的 mcpack2pb](https://github.com/apache/incubator-brpc/tree/master/src/mcpack2pb)里也是类似的问题。

或者是数据存储/分发的场景，对于接收的 message，我们希望有的字段能够覆盖写、有的删除、有的建索引，如果在定义 message 的时候就能够提前约定，策略同学在新增字段的时候就可以直接实现对存储的预期。

protobuf 的 [Custom Options](https://developers.google.com/protocol-buffers/docs/proto#customoptions)特性可以实现这点。

## 2. descriptor

protobuf 里实现了众多的[Descriptor类型](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/descriptor.h)，例如`Descriptor`对应具体的 message，`FieldDescriptor`对应某一个字段，`EnumDescriptor`，`FileDescriptor`,`ServiceDescriptor`等都对应了 protobuf 里的某个"实体"，用于支持我们获取其描述性的结构性质。

对于每种`xxxDescriptor`，在[descritpor.proto](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/descriptor.proto)都有相应的 Options，例如`MessageOptions` `FieldOptions` `EnumOptions` `ServiceOptions`，这些都定义为普通的 pb message.

这些类之间的关系，具体的，举个例子：

```cpp
class LIBPROTOBUF_EXPORT Descriptor {
    ...
    const MessageOptions& options() const;
    ...
    // The number of fields in this message type.
    int field_count() const;
    // Gets a field by index, where 0 <= index < field_count().
    // These are returned in the order they were defined in the .proto file.
    const FieldDescriptor* field(int index) const;

class LIBPROTOBUF_EXPORT FieldDescriptor {
    ...
    const FieldOptions& options() const;
```
我们可以通过`Descriptor`获取到字段的`FieldOptions`，而 Custom Options 的奥秘，就来自于对这些自定义 message options 的扩展，接下来，看个具体`extend FieldOptions`的示例，其他也是类似。

## 3. extend FieldOptions 表示不同类型

`FieldOptions`定义为 message:

```
message FieldOptions {
    ...
    optional bool deprecated = 3 [default=false];
    ...
    // Clients can define custom options in extensions of this message. See above.
    extensions 1000 to max;
}
```

我们通过扩展`FieldOptions`指定该 field 类型是与 pb 原类型语义相同(ORIGINAL)，还是需要指定为 RAW 类型(针对 RAW 类型，我们需要有不同的处理方式)。

```
enum CustomFieldType {
    RAW = 0;
    ORIGINAL = 1;
}

extend google.protobuf.FieldOptions {
    optional CustomFieldType custom_field_type = 50002 [default = ORIGINAL];
}
```

注意这里只是简单的 extensions，因此可以正常使用`GetExtension`等接口获取该值。

接下来我们定义自己的 message:

```
message MyMessage {
    optional int32 old_field = 1 [deprecated=true];
    optional int32 new_field = 2;
    optional string name = 3;
    optional string raw_str = 4 [(custom_field_type) = RAW];
}
```

`old_field` `new_field`用于对比`FieldOptions`预定义的`deprecated`.  
`raw_str`则定义了`(custom_field_type) = RAW`，由于该值默认为`ORIGINAL`，定义了`name`用于对比。  

接下来就是具体使用的例子：

```cpp
    const ::google::protobuf::Descriptor* descriptor = MyMessage::descriptor();
    for (int i = 0; i < descriptor->field_count(); ++i) {
        const ::google::protobuf::FieldDescriptor* field_descriptor = descriptor->field(i);
        const ::google::protobuf::FieldOptions& field_options = field_descriptor->options();
        std::cout << "FieldName:" << field_descriptor->full_name()
            << "\tCustomFieldType:" << field_options.GetExtension(custom_field_type)
            << "\tDeprecated:" << field_options.deprecated() << std::endl;
    }
```

输出为：

```cpp
FieldName:MyMessage.old_field   CustomFieldType:1       Deprecated:1
FieldName:MyMessage.new_field   CustomFieldType:1       Deprecated:0
FieldName:MyMessage.name        CustomFieldType:1       Deprecated:0
FieldName:MyMessage.raw_str     CustomFieldType:0       Deprecated:0
```

可以看到，通过这种方式，即使都是`optional string`类型，我们也可以通过自定义的 options，来区分更细的不同类型(这里是 RAW)，之后执行不同的操作，例如对于普通的 string 类型，mcpack 是`put_str/get_str`，而对于 RAW，则需要`put_raw/get_raw`。

`deprecated`作为`message FieldOptions`的预定义 field，可以直接使用`deprecated()`接口，而`custom_field_type`作为 extend 的字段，需要使用`GetExtension()`接口，这点与普通 message 是一致的。

## 4. TIPS

自定义的类型使用时需要使用`(...)`括起来，以跟预定义 option field 区分。扩展的字段范围可以直接参考 descriptor.proto，大部分是`extensions 1000 to max;`，跟普通的 extensions 一样，在项目里尽量避免冲突。
