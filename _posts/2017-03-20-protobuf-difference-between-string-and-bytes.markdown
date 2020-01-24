---
title: protobuf之string bytes的区别
date: 2017-3-20 16:24:34
excerpt: "protobuf之string bytes的区别"
tags: protobuf, string  bytes
---

protobuf提供了多种基础数据格式，包括string/bytes。从字面意义上，我们了解bytes适用于任意的二进制字节序列。然而对C++程序员来讲，std::string既能存储ASCII文本字符串，也能存储任意多个`\0`的二进制序列。那么区别在哪里呢？

同时在实际使用中，我们偶尔会看到类似这样的运行错误：

```
[libprotobuf ERROR google/protobuf/wire_format.cc:1091] String field 'str' contains invalid UTF-8 data when serializing a protocol buffer. Use the 'bytes' type if you intend to send raw bytes. 
[libprotobuf ERROR google/protobuf/wire_format.cc:1091] String field 'str' contains invalid UTF-8 data when parsing a protocol buffer. Use the 'bytes' type if you intend to send raw bytes. 
```

这篇文章从源码角度分析下`string/bytes`类型的区别。

<!--more-->

在[之前的文章](http://izualzhy.cn/protobuf-encoding)里介绍过protobuf序列化的过程，我们看下`string/bytes`序列化的过程。

所有的序列化操作都会在`SerializeFieldWithCachedSizes`这个函数里进行。根据不同的类型调用对应的序列化函数，例如对于`string`类型

```
      case FieldDescriptor::TYPE_STRING: {
        string scratch;
        const string& value = field->is_repeated() ?
          message_reflection->GetRepeatedStringReference(
            message, field, j, &scratch) :
          message_reflection->GetStringReference(message, field, &scratch);
        VerifyUTF8StringNamedField(value.data(), value.length(), SERIALIZE,
                                   field->name().c_str());
        WireFormatLite::WriteString(field->number(), value, output);
        break;
      }
```

而对于`bytes`类型：

```
      case FieldDescriptor::TYPE_BYTES: {
        string scratch;
        const string& value = field->is_repeated() ?
          message_reflection->GetRepeatedStringReference(
            message, field, j, &scratch) :
          message_reflection->GetStringReference(message, field, &scratch);
        WireFormatLite::WriteBytes(field->number(), value, output);
        break;
      }
```

可以看到在序列化时主要有两点区别：

1. `string`类型调用了`VerifyUTF8StringNamedField`函数  
2. 序列化函数不同：`WriteString vs WriteBytes`

关于第二点，两个函数都定义在`wire_format_lite.cc`，实现是相同的。

那么我们继续看下第一点，`VerifyUTF8StringNamedField`调用了`VerifyUTF8StringFallback`（话说一直不理解fallback在这里什么意思，protobuf源码里经常看到这个后缀）。看下这个函数的实现：

```

void WireFormat::VerifyUTF8StringFallback(const char* data,
                                          int size,
                                          Operation op,
                                          const char* field_name) {
  if (!IsStructurallyValidUTF8(data, size)) {
    const char* operation_str = NULL;
    switch (op) {
      case PARSE:
        operation_str = "parsing";
        break;
      case SERIALIZE: 
        operation_str = "serializing";
        break;
      // no default case: have the compiler warn if a case is not covered.
    }
    string quoted_field_name = "";
    if (field_name != NULL) {
      quoted_field_name = StringPrintf(" '%s'", field_name);
    }
    // no space below to avoid double space when the field name is missing.
    GOOGLE_LOG(ERROR) << "String field" << quoted_field_name << " contains invalid "
               << "UTF-8 data when " << operation_str << " a protocol "
               << "buffer. Use the 'bytes' type if you intend to send raw "
               << "bytes. ";
  }
}
```

运行错误是从这里输出的，关键还是在于`IsStructurallyValidUTF8`这个函数，实现在`structurally_valid.cc`里：

```
bool IsStructurallyValidUTF8(const char* buf, int len) {
  if (!module_initialized_) return true;
  
  int bytes_consumed = 0;
  UTF8GenericScanFastAscii(&utf8acceptnonsurrogates_obj,
                           buf, len, &bytes_consumed);
  return (bytes_consumed == len);
}
```

这里逐个字符扫描是否符合utf-8规范，比如`110xxxxx 10xxxxxx`这样，具体可以参考utf-8的编码标准。

反序列化过程类似。

看到这里我们可以得到这样的结论：

1. protobuf里的`string/bytes`在C++接口里实现上都是`std::string`。  
2. 两者序列化、反序列化格式上一致，不过对于`string`格式，会有一个utf-8格式的检查。

出于效率，我们应当在确定字段编码格式后直接使用`bytes`，减少utf8编码的判断，效率上会有提高。

注意以上代码在pb2.6下，2.4不会输出`field_name`。

据了解`java`接口上有一定的区别，分别对应`String`以及`ByteString`。
