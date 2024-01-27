---
title:  "protobuf反射详解"
date: 2016-04-22 15:19:03
excerpt: "protobuf反射详解"
tags: protobuf
---

本文主要介绍protobuf里的反射功能，使用的pb版本为2.6.1，同时为了简洁，对repeated/extension字段的处理方法没有说明。

最初是起源于这样一个问题：  
**给定一个pb对象，如何自动遍历该对象的所有字段？**  

即是否有一个通用的方法可以遍历任意pb对象的所有字段，而不用关心具体对象类型。    
使用场景上有很多：  
比如json格式字符串的相互转换，或者bigtable里根据pb对象的字段自动写列名和对应的value。

<!--more-->

例如定义了pb messge类型Person如下：

```cpp
Person person;
person.set_name("yingshin");
person.set_age(21);
```

能否将该对象自动转化为json字符串`{"name":"yingshin","age":21}`,或者自动的写hbase里的多列：  

|key          |column-name        |column-value         |
|-------------|-------------------|---------------------|
|uid          |name               |yingshin             |
|uid          |age                |21                   |

如果设置了新的字段，比如`person.set_email("izualzhy.cn/com")`，则自动添加新的一列：  

|key          |column-name        |column-value         |
|-------------|-------------------|---------------------|
|uid          |email              |izualzhy.cn/com     |

答案就是 **pb的反射功能**。

我们的目标是提供这样两个接口：  

```cpp
//从给定的message对象序列化为固定格式的字符串
void serialize_message(const google::protobuf::Message& message, std::string* serialized_string);
//从给定的字符串按照固定格式还原为原message对象
void parse_message(const std::string& serialized_string, google::protobuf::Message* message);
```


接下来逐步介绍下如何实现。

#### 1. 反射相关接口

要介绍pb的反射功能，先看一个相关的UML示例图：

![pb-reflection](/assets/images/pb-reflection.png)

各个类以及接口说明:

##### 1.1 Message

Person是自定义的pb类型，继承自Message.  MessageLite作为Message基类，更加轻量级一些。  
通过Message的两个接口`GetDescriptor/GetReflection`，可以获取该类型对应的Descriptor/Reflection。  

##### 1.2 Descriptor

Descriptor是对message类型定义的描述，包括message的名字、所有字段的描述、原始的proto文件内容等。  
本文中我们主要关注跟字段描述相关的接口，例如：  

1. 获取所有字段的个数：`int field_count() const`  
2. 获取单个字段描述类型`FieldDescriptor`的接口有很多个，例如

```cpp
const FieldDescriptor* field(int index) const;//根据定义顺序索引获取
const FieldDescriptor* FindFieldByNumber(int number) const;//根据tag值获取
const FieldDescriptor* FindFieldByName(const string& name) const;//根据field name获取
```

##### 1.3 FieldDescriptor

FieldDescriptor描述message中的单个字段，例如字段名，字段属性(optional/required/repeated)等。  
对于proto定义里的每种类型，都有一种对应的C++类型，例如：  

```cpp
enum CppType {
	CPPTYPE_INT32 = 1, //TYPE_INT32, TYPE_SINT32, TYPE_SFIXED32
}
```
获取类型的label属性： 

```cpp
enum Label {
	LABEL_OPTIONAL = 1, //optional
	LABEL_REQUIRED = 2, //required
	LABEL_REPEATED = 3, //repeated

	MAX_LABEL = 3, //Constant useful for defining lookup tables indexed by Label.
}
```
获取字段的名称:`const string& name() const;`。  

##### 1.4 Reflection
Reflection主要提供了动态读写pb字段的接口，对pb对象的自动读写主要通过该类完成。  
对每种类型，Reflection都提供了一个单独的接口用于读写字段对应的值。  

例如对于读操作：

```cpp
  virtual int32  GetInt32 (const Message& message,
                           const FieldDescriptor* field) const = 0;
  virtual int64  GetInt64 (const Message& message,
                           const FieldDescriptor* field) const = 0;
```
特殊的，对于枚举和嵌套的message：

```cpp
  virtual const EnumValueDescriptor* GetEnum(
      const Message& message, const FieldDescriptor* field) const = 0;
  // See MutableMessage() for the meaning of the "factory" parameter.
  virtual const Message& GetMessage(const Message& message,
                                    const FieldDescriptor* field,
                                    MessageFactory* factory = NULL) const = 0;
```

对于写操作也是类似的接口，例如`SetInt32/SetInt64/SetEnum`等。  

#### 2. 反射示例

示例主要是接收任意类型的message对象，遍历解析其中的每个字段、以及对应的值，按照自定义的格式存储到一个string中。同时重新反序列化该string，读取字段以及value，填充到message对象中。例如：  

其中Person是自定义的protobuf message类型，用于设置一些字段验证我们的程序。  
单纯的序列化/反序列化功能可以通过pb自带的SerializeToString/ParseFromString接口完成。这里主要是为了同时展示**自动从pb对象里提取field/value，自动根据field/value来还原pb对象**这个功能。

```cpp
int main() {
    std::string serialized_string;
    {
        tutorial::Person person;
        person.set_name("yingshin");
        person.set_id(123456789);
        person.set_email("zhy198606@gmail.com");
        ::tutorial::Person_PhoneNumber* phone = person.mutable_phone();
        phone->set_type(tutorial::Person::WORK);
        phone->set_number("13266666666");

        serialize_message(person, &serialized_string);
    }

    {
        tutorial::Person person;
        parse_message(serialized_string, &person);
    }

    return 0;
}
```

其中Person定义是对example里的addressbook.proto做了少许修改(修改的原因是本文没有涉及pb里数组的处理)

```cpp
package tutorial;

message Person {
  required string name = 1;
  required int32 id = 2;        // Unique ID number for this person.
  optional string email = 3;

  enum PhoneType {
      MOBILE = 0;
      HOME = 1;
      WORK = 2;
  }

  message PhoneNumber {
      required string number = 1;
      optional PhoneType type = 2 [default = HOME];
  }

  optional PhoneNumber phone = 4;
}
```

#### 3. 反射实例实现

##### 3.1 serialize_message
serialize_message遍历提取message中各个字段以及对应的值，序列化到string中。  
主要思路就是通过Descriptor得到每个字段的描述符：字段名、字段的cpp类型。  
通过Reflection的GetX接口获取对应的value。  

```cpp
void serialize_message(const google::protobuf::Message& message, std::string* serialized_string) {
    const google::protobuf::Descriptor* descriptor = message.GetDescriptor();
    const google::protobuf::Reflection* reflection = message.GetReflection();

    for (int i = 0; i < descriptor->field_count(); ++i) {
        const google::protobuf::FieldDescriptor* field = descriptor->field(i);
        bool has_field = reflection->HasField(message, field);

        if (has_field) {
            //arrays not supported
            assert(!field->is_repeated());

            switch (field->cpp_type()) {
#define CASE_FIELD_TYPE(cpptype, method, valuetype)\
                case google::protobuf::FieldDescriptor::CPPTYPE_##cpptype:{\
                    valuetype value = reflection->Get##method(message, field);\
                    int wsize = field->name().size();\
                    serialized_string->append(reinterpret_cast<char*>(&wsize), sizeof(wsize));\
                    serialized_string->append(field->name().c_str(), field->name().size());\
                    wsize = sizeof(value);\
                    serialized_string->append(reinterpret_cast<char*>(&wsize), sizeof(wsize));\
                    serialized_string->append(reinterpret_cast<char*>(&value), sizeof(value));\
                    break;\
                }

                CASE_FIELD_TYPE(INT32, Int32, int);
                CASE_FIELD_TYPE(UINT32, UInt32, uint32_t);
                CASE_FIELD_TYPE(FLOAT, Float, float);
                CASE_FIELD_TYPE(DOUBLE, Double, double);
                CASE_FIELD_TYPE(BOOL, Bool, bool);
                CASE_FIELD_TYPE(INT64, Int64, int64_t);
                CASE_FIELD_TYPE(UINT64, UInt64, uint64_t);
#undef CASE_FIELD_TYPE
                case google::protobuf::FieldDescriptor::CPPTYPE_ENUM: {
                    int value = reflection->GetEnum(message, field)->number();
                    int wsize = field->name().size();
                    //写入name占用字节数
                    serialized_string->append(reinterpret_cast<char*>(&wsize), sizeof(wsize));
                    //写入name
                    serialized_string->append(field->name().c_str(), field->name().size());
                    wsize = sizeof(value);
                    //写入value占用字节数
                    serialized_string->append(reinterpret_cast<char*>(&wsize), sizeof(wsize));
                    //写入value
                    serialized_string->append(reinterpret_cast<char*>(&value), sizeof(value));
                    break;
                }
                case google::protobuf::FieldDescriptor::CPPTYPE_STRING: {
                    std::string value = reflection->GetString(message, field);
                    int wsize = field->name().size();
                    serialized_string->append(reinterpret_cast<char*>(&wsize), sizeof(wsize));
                    serialized_string->append(field->name().c_str(), field->name().size());
                    wsize = value.size();
                    serialized_string->append(reinterpret_cast<char*>(&wsize), sizeof(wsize));
                    serialized_string->append(value.c_str(), value.size());
                    break;
                }
                case google::protobuf::FieldDescriptor::CPPTYPE_MESSAGE: {
                    std::string value;
                    int wsize = field->name().size();
                    serialized_string->append(reinterpret_cast<char*>(&wsize), sizeof(wsize));
                    serialized_string->append(field->name().c_str(), field->name().size());
                    const google::protobuf::Message& submessage = reflection->GetMessage(message, field);
                    serialize_message(submessage, &value);
                    wsize = value.size();
                    serialized_string->append(reinterpret_cast<char*>(&wsize), sizeof(wsize));
                    serialized_string->append(value.c_str(), value.size());
                    break;
                }
            }
        }
    }
}
```

##### 3.2 parse_message
parse_message通过读取field/value，还原message对象。  
主要思路跟serialize_message很像，通过Descriptor得到每个字段的描述符FieldDescriptor，通过Reflection的SetX填充message。  

```cpp
void parse_message(const std::string& serialized_string, google::protobuf::Message* message) {
    const google::protobuf::Descriptor* descriptor = message->GetDescriptor();
    const google::protobuf::Reflection* reflection = message->GetReflection();
    std::map<std::string, const google::protobuf::FieldDescriptor*> field_map;

    for (int i = 0; i < descriptor->field_count(); ++i) {
        const google::protobuf::FieldDescriptor* field = descriptor->field(i);
        field_map[field->name()] = field;
    }

    const google::protobuf::FieldDescriptor* field = NULL;
    size_t pos = 0;
    while (pos < serialized_string.size()) {
        int name_size = *(reinterpret_cast<const int*>(serialized_string.substr(pos, sizeof(int)).c_str()));
        pos += sizeof(int);
        std::string name = serialized_string.substr(pos, name_size);
        pos += name_size;

        int value_size = *(reinterpret_cast<const int*>(serialized_string.substr(pos, sizeof(int)).c_str()));
        pos += sizeof(int);
        std::string value = serialized_string.substr(pos, value_size);
        pos += value_size;

        std::map<std::string, const google::protobuf::FieldDescriptor*>::iterator iter =
            field_map.find(name);
        if (iter == field_map.end()) {
            fprintf(stderr, "no field found.\n");
            continue;
        } else {
            field = iter->second;
        }

        assert(!field->is_repeated());
        switch (field->cpp_type()) {
#define CASE_FIELD_TYPE(cpptype, method, valuetype)\
            case google::protobuf::FieldDescriptor::CPPTYPE_##cpptype: {\
                reflection->Set##method(\
                        message,\
                        field,\
                        *(reinterpret_cast<const valuetype*>(value.c_str())));\
                std::cout << field->name() << "\t" << *(reinterpret_cast<const valuetype*>(value.c_str())) << std::endl;\
                break;\
            }
            CASE_FIELD_TYPE(INT32, Int32, int);
            CASE_FIELD_TYPE(UINT32, UInt32, uint32_t);
            CASE_FIELD_TYPE(FLOAT, Float, float);
            CASE_FIELD_TYPE(DOUBLE, Double, double);
            CASE_FIELD_TYPE(BOOL, Bool, bool);
            CASE_FIELD_TYPE(INT64, Int64, int64_t);
            CASE_FIELD_TYPE(UINT64, UInt64, uint64_t);
#undef CASE_FIELD_TYPE
            case google::protobuf::FieldDescriptor::CPPTYPE_ENUM: {
                const google::protobuf::EnumValueDescriptor* enum_value_descriptor =
                    field->enum_type()->FindValueByNumber(*(reinterpret_cast<const int*>(value.c_str())));
                reflection->SetEnum(message, field, enum_value_descriptor);
                std::cout << field->name() << "\t" << *(reinterpret_cast<const int*>(value.c_str())) << std::endl;
                break;
            }
            case google::protobuf::FieldDescriptor::CPPTYPE_STRING: {
                reflection->SetString(message, field, value);
                std::cout << field->name() << "\t" << value << std::endl;
                break;
            }
            case google::protobuf::FieldDescriptor::CPPTYPE_MESSAGE: {
                google::protobuf::Message* submessage = reflection->MutableMessage(message, field);
                parse_message(value, submessage);
                break;
            }
            default: {
                break;
            }
        }
    }
}
```
