---
title: "boost容器之dynamic_bitset"
date: 2017-12-09 21:39:48
excerpt: "boost容器之dynamic_bitset"
tags: boost
---

boost提供了很多非常有用的容器及数据结构，是对stl的一个极大的增强。本文主要介绍二进制数值处理的**boost::dynamic_bitset**

<!--more-->

C++标准提供了两个二进制数值处理工具:`vector<bool>` 和 `bitset`

`vector<bool>`是`vector`的特化，并不是单纯存储了`bool`类型的vector，可以动态增长，但是没有很好的位运算的接口。

`bitset`支持很方便的位操作，但是不能支持动态增长。

`boost::dynamic_bitset`则提供了丰富的位运算，同时支持动态增长。

## 1. 构造

`dynamic_bitset`是一个模板类

```cpp
template <typename Block, typename Allocator>
class dynamic_bitset
```

Block是一个类型参数，表示以什么类型存储二进制位，必须是一个unsigned类型，默认unsigned long。

构造函数可以直接传入01字符串，也可以传入一个整数。

```cpp
#include <iostream>
#include "boost/dynamic_bitset.hpp"
#include "boost/utility/binary.hpp"

int main() {
    {
        // error: invalid conversion from 'const char*' to 'boost::dynamic_bitset<>::size_type {aka long unsigned int}' [-fpermissive]
        // boost::dynamic_bitset<> db("0100");
        boost::dynamic_bitset<> db(std::string("0100"));
        std::cout << db << std::endl;
    }

    {
        boost::dynamic_bitset<> db1(3, 5);//101
        std::cout << db1 << std::endl;

        boost::dynamic_bitset<> db2(0x16, BOOST_BINARY(10101));//0000000000000000010101
        std::cout << db2 << std::endl;
    }

    return 0;
}
```

注意字符串构造时传入01字符串，否则会在运行时报错。同时不能直接传入C字符串，否则编译报错

```cpp
        // error: invalid conversion from 'const char*' to 'boost::dynamic_bitset<>::size_type {aka long unsigned int}' [-fpermissive]
```

整数构造时第一个参数表示大小，第二个参数表示值，如果要传入二进制数值，可以使用`BOOST_BINARY`。第二个参数没有设置时，默认全部bit为0。

注意如果传入非01数值，会在编译时报错。

## 2. 常用接口

### 2.1 大小

`resize/clear size/empty`接口用于调整和获取大小

```cpp
    boost::dynamic_bitset<> db(0x16, BOOST_BINARY(10101));;
    std::cout << db << "\t" << db.size() << std::endl;//0000000000000000010101  22

    db.resize(3);
    std::cout << db << "\t" << db.size() << std::endl;//101     3

    db.clear();
    std::cout << db.empty() << std::endl;//1
```

注意resize后的大小如果变小，那么高位的二进制位被删除。如果变大，可以通过第二个参数指定扩展的值，默认为0。

### 2.2 追加数据

`push_back`用于向**高位**添加数据

```cpp
    boost::dynamic_bitset<> db(0x16, BOOST_BINARY(10101));;
    std::cout << db << "\t" << db.size() << std::endl;//0000000000000000010101  22

    db.push_back(1);
    std::cout << db << "\t" << db.size() << std::endl;//10000000000000000010101 23
```

`append`用于追加一个整数，不过注意整数会转化为一整个Block，在64位系统上占64bits.

```cpp
    boost::dynamic_bitset<> db(0x16, BOOST_BINARY(10101));;
    db.append(5);//append 101
    std::cout << db << std::endl;//00000000000000000000000000000000000000000000000000000000000001010000000000000000010101
```

### 2.3 读写

如果读写已知的位，也可以直接使用`operator[]`。

```cpp
    boost::dynamic_bitset<> db(5, BOOST_BINARY(10100));
    std::cout << db << std::endl;
    //test检验第n位是否是1
    std::cout << db.test(0) << std::endl;//0
    std::cout << db.test(0, 0) << std::endl;//1
    //如果有一个1，那么any返回true
    std::cout << db.any() << "\t" << boost::dynamic_bitset<>(5).any() << std::endl;//1       0
    //如果不存在1，那么none返回true
    std::cout << db.none() << "\t" << boost::dynamic_bitset<>(5).none() << std::endl;/ 0       1
    //count用于统计1出现的次数
    std::cout << db.count() << std::endl;//2
```

`set/reset/flip`用于操作所有的二进制位

```cpp
    boost::dynamic_bitset<> db(5, BOOST_BINARY(10100));
    std::cout << db << std::endl;
    //flip反转
    db.flip();
    std::cout << db << std::endl;//01011
    //set设置为1
    db.set();
    std::cout << db << std::endl;//11111
    //reset设置为0
    db.reset();
    std::cout << db << std::endl;//00000
```

当然也可以使用带参数的版本修改某个位

```cpp
    // basic bit operations
    dynamic_bitset& set(size_type n, bool val = true);
    dynamic_bitset& set();
    dynamic_bitset& reset(size_type n);
    dynamic_bitset& reset();
    dynamic_bitset& flip(size_type n);
    dynamic_bitset& flip();
```

也可以转为`unsigned long`或者`std::string`

```cpp
    boost::dynamic_bitset<> db;
    db.push_back(false);
    db.push_back(false);
    db.push_back(true);
    db.push_back(true);

    std::cout << db.to_ulong() << std::endl;//12
    std::string s;
    boost::to_string(db, s);
    std::cout << s << std::endl;//1100
```

### 2.4 查找

`find_first`从下标0开始，查找第一个值为1的位置。  
`find_next(pos)`从下标pos开始，查找第一个值为1的位置。  

```cpp
    // lookup
    size_type find_first() const;
    size_type find_next(size_type pos) const;
```

```cpp
    boost::dynamic_bitset<> db(std::string("110100"));
    std::cout << db << std::endl;

    for (size_t i = db.find_first(); i != boost::dynamic_bitset<>::npos;) {
        std::cout << i << " ";//2 4 5
        i = db.find_next(i);
    }
```

### 2.5 集合操作

```cpp
    boost::dynamic_bitset<> db1(5, BOOST_BINARY(10101));
    boost::dynamic_bitset<> db2(5, BOOST_BINARY(10010));

    std::cout << (db1 | db2) << std::endl;//10111
    std::cout << (db1 & db2) << std::endl;//10000
    std::cout << (db1 - db2) << std::endl;//00101

    boost::dynamic_bitset<> db3(5, BOOST_BINARY(101));
    std::cout << db3.is_proper_subset_of(db1) << std::endl;//1

    boost::dynamic_bitset<> db4(db2);
    std::cout << db4.is_subset_of(db2) << std::endl;//1
    std::cout << db4.is_proper_subset_of(db2) << std::endl;//0
```

## 3. 总结

`bitset`的实现虽然不算复杂，但是使用`boost::dynamic_bitset`，在接口丰富度、稳定性、开发效率上都比我们自己封装一个要好很多。我在实际应用时，每次处理一个数组，需要遍历判断部分元素是否需要回发到上游模块，通过标记到一个`dynamic_bitset`，之后内部处理时跳过值为1的元素，这样比对数组删除某些元素性能要高一些。

## 4. 参考

[Boost程序库完全开发指南](https://book.douban.com/subject/26320630/)
