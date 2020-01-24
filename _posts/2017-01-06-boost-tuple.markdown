---
title: "boost容器之tuple"
date: 2018-01-06 22:22:05
excerpt: "boost容器之tuple"
tags: boost  tuple
---

python里tuple(元组)的使用很常见，例如

```python
def foo():
    return (1, 'a', 1.1)
```

相比定义一个class要考虑：命名、代码长度、可读性等，使用起来更加灵活。

C++里我们常用的替代方案是`std::pair`，例如protobuf源码里

```cpp
for (;;) {
    ::std::pair< ::google::protobuf::uint32, bool> p = input->ReadTagWithCutoffNoLastTag(127u);
    ...
```

通过`std::pair::first/second`接口分别获取需要的返回值。但是`std::pair`只能提供两个返回值，相比python里的tuple可用性还是差了一些，本文要介绍的就是加入到C++11的`boost::tuple`，也就是C++里的元组。

灵活使用`boost::tuple`，能够很大程度上减少代码量，避免许多一次性的`struct/class`定义。

<!--more-->

## 1. 使用

先看一个使用的例子：

```cpp
#include <string>
#include <iostream>
#include "boost/tuple/tuple.hpp"
#include "boost/tuple/tuple_io.hpp"

int main() {
    typedef boost::tuple<std::string, int> animal;
    animal cat("cat", 4);

    //(cat 4)
    std::cout << cat << std::endl;

    return 0;
}
```

`boost::tuple`定义在boost/tuple/tuple.hpp，是一个模板类。

同时支持`operator <<`（只要各元素支持），需要include "boost/tuple/tuple_io.hpp"。

可以看到使用上跟`std::pair`很像，其实可以认为`std::pair`是只传入两个参数下的tuple，而tuple最多支持10个类型参数。

## 2. 访问

get接口用于访问tuple内各个元素

```cpp
    boost::tuple<std::string, std::string, int> person("Jeff Dean", "Google", 1);
    //(Jeff Dean Google 1)
    std::cout << person << std::endl;
    //Jeff Dean
    std::cout << person.get<0>() << std::endl;
    //Google
    std::cout << person.get<1>() << std::endl;
    //1
    std::cout << person.get<2>() << std::endl;
```

注意`get`是一个模板函数，调用方式为`get<N>()`，返回的是一个reference，可以用来修改元素。具体可以参考`make_tuple`一节。

因为模板实例是需要在编译时确定的，如果要遍历所有元素，也就不能采用下面这种方式

```cpp
//compile error
    for (int i = 0; i < 3; ++i) {
        std::cout << person.get<i>() << std::endl;
    }
```

## 3. 遍历

tuple内部实现是一个链表，通过`get_head get_tail`可以获取链表的头尾部分。通过这种方式可以实现遍历

```cpp
template <typename Tuple>
void print(const Tuple& t) {
    std::cout << t.get_head() << ",";
    print(t.get_tail());
}

template<>
void print(const boost::tuples::null_type&) {
    std::cout << std::endl;
}

...
print(person);
```

gdb看下`head tail`，结构更清楚些：

```gdb
(gdb) p person
$1 = {
  <boost::tuples::cons<std::basic_string<char, std::char_traits<char>, std::allocator<char> >, boost::tuples::cons<std::basic_string<char, std::char_traits<char>, std::allocator<char> >, boost::tuples::cons<int, boost::tuples::null_type> > >> = {
    head = "Jeff Dean",
    tail = {
      head = "Google",
      tail = {
        head = 1
      }
    }
  }, <No data fields>}
(gdb) p person.head
$2 = "Jeff Dean"
(gdb) p person.tail.head
$3 = "Google"
(gdb) p person.tail.tail.head
$4 = 1
```

## 4. make_tuple

与`make_pair`类似，tuple可以使用`make_tuple`来创建。

```cpp
    //注意如果要修改元素，std::string是必须的
    auto person = boost::make_tuple(std::string("Jeff Dean"),
            std::string("Google"),
            1);
    //(Jeff Dean Google 1)
    std::cout << person << std::endl;

    person.get<1>() = "Bidu:?";
    //(Jeff Dean Bidu:? 1)
    std::cout << person << std::endl;
```

## 5. tie

`boost::tie`提供了把变量绑定到tuple的方式，创建一个引用的tuple。

```cpp
    std::string company = "Google";
    boost::tie(company, boost::tuples::ignore).get<0>() = "Bidu:?";
    //Bidu:?
    std::cout << company << std::endl;```
```

可以看到通过boost::tie产生的临时对象修改元素0后，变量`company`发生了变化。

这里boost::tie实际上是生成了一个`tuple<std::string&, ...>`的tuple类型，注意boost还提供了`boost::tuples::ignore`来忽略不需要的变量，类似于go里的`_`。

## 6. 参考

1. [Boost程序库完全开发指南](https://book.douban.com/subject/26320630/)  
2. [Boost.Tuple](https://theboostcpplibraries.com/boost.tuple)  
