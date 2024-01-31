---
title: "boost容器之circular_buffer"
date: 2017-12-10 16:02:46
excerpt: "boost容器之circular_buffer"
tags: boost
---

本文主要介绍boost里的环形缓冲区**boost::circular_buffer**

之前接手过一个模块，模块内需要记录某张表最近12次的数据，这种情况就需要有一个可以“循环利用空间”的数据结构，当空间满时，新数据自动替换掉老数据的内容，这就是`boost::circular_buffer`的作用。

可以把`boost::circular_buffer`想象成一个环，首尾相接，当元素数量达到上限时，从首部自动开始重用之前的空间。

<!--more-->

## 1. 构造

`circular_buffer`的构造比较简单，传入的参数为环形缓冲区的空间大小。模板类型为存储的元素类型。

```
#include <iostream>
#include "boost/circular_buffer.hpp"

int main() {
    boost::circular_buffer<int> c(10);
    std::cout << c.size() << "\t" << c.capacity() << std::endl;//0   10

    return 0;
}
```

## 2. 使用

使用上`circular_buffer`最常用的是从两端写入数据，也支持数据的`insert erase`

```
#include <iostream>
#include "boost/circular_buffer.hpp"

template<class T>
void print(const T& container) {
    for (auto& x : container) {
        std::cout << x << " ";
    }
    std::cout << "full:" << container.full()
        << "\tsize:" << container.size()
        << "\tcapacity:" << container.capacity()
        << "\tis_linearized:" << container.is_linearized()
        << std::endl;
}

int main() {
    const int cb_max_size = 5;
    boost::circular_buffer<int> cb(cb_max_size);

    for (int i = 1; i <= cb_max_size; ++i) {
        cb.push_back(i);
    }
    // 1 2 3 4 5 full:1        size:5  capacity:5      is_linearized:1
    print(cb);
    cb.push_back(6);
    //   2 3 4 5 6 full:1        size:5  capacity:5      is_linearized:0
    print(cb);

    cb.push_back(7);
    //     3 4 5 6 7 full:1        size:5  capacity:5      is_linearized:0
    print(cb);

    cb.pop_back();
    //     3 4 5 6 full:0  size:4  capacity:5      is_linearized:0
    print(cb);

    cb.pop_front();
    //       4 5 6 full:0    size:3  capacity:5      is_linearized:0
    print(cb);

    cb.rotate(cb.begin() + 2);
    // 6 4 5 full:0    size:3  capacity:5      is_linearized:1
    print(cb);

    cb.set_capacity(10);
    // 6 4 5 full:0    size:3  capacity:10     is_linearized:1
    print(cb);

    return 0;
}
```

`circular_buffer`的使用非常简单，上面的代码包含了大部分用法。

注意`linearize`可以把缓冲区线性化成一个特殊的普通数据，`is_linearized`可以检测缓冲区可否线性化，即`cb.begin()`正好是缓冲区内存开始的位置。

## 3. 参考

[Boost程序库完全开发指南](https://book.douban.com/subject/26320630/)
