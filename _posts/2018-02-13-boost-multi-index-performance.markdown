---
title: "boost容器之multi_index性能"
date: 2018-02-13 14:56:10
excerpt: "boost容器之multi_index性能"
tags: boost  multi_index
---

上次介绍了boost里的[multi_index_container](http://izualzhy.cn/boost-multi-index)，通过组合不同stl容器，实用性很强。

例如对于[LRU](https://en.wikipedia.org/wiki/Cache_replacement_policies#LRU) cache，[代码量](https://github.com/yingshin/Tiny-Tools/blob/master/cache/multi_index_lru.cp)能够得到很大精简，但是相信大部分程序员（包括我）都有一颗造轮子的心，直接使用**multi_index_container**性能如何？

本文尝试探索下**multi_index_container**的性能。

<!--more-->

## 1. boost里的performance

### 1.1. emulate_std_containers

**multi_index_container**支持的索引个数是没有限制的，也就是说退化为单索引的情况下，我们可以模拟stl的标准容器。

例如对于`std::set`

```cpp
std::set<Key,Compare,Allocator>
->
multi_index_container<
  Key,
  indexed_by<ordered_unique<identity<Key>,Compare> >,
  Allocator
>
```

默认情况(Compare=std::less<Key> and Allocator=std::allocator<Key>)，我们可以得到简化后的这个对应关系：

```cpp
std::set<Key> -> multi_index_container<Key>
```

也就是上篇笔记开始提到的单索引的例子。

`std::multi_set`也可以类似的推导关系：

```cpp
std::multiset<Key>
->
multi_index_container<
  Key,
  indexed_by<ordered_non_unique<identity<Key> > >
>
```

对于`std::map std::multimap std::list`都可以使用`multi_index_container`得到类似的数据结构，具体可以参考[Emulating standard containers with multi_index_container
](http://www.boost.org/doc/libs/1_66_0/libs/multi_index/doc/tutorial/techniques.html#emulate_std_containers)

### 1.2. performance

实际上[boost官方文档](http://www.boost.org/doc/libs/1_66_0/libs/multi_index/doc/performance.html)里给了一部分性能比较的思路和实战。其中一部分就是通过上面的定义直接对比stl里的containers.

性能文档首先给了一个多索引的例子(当然这个例子感觉很不恰当，这是啥😱)

```cpp
typedef multi_index_container<
  int,
  indexed_by<
    ordered_unique<identity<int> >,
    ordered_non_unique<identity<int>, std::greater >,
  >
> indexed_t;
```
索引1对数据升序排列，索引2对数据降序排列（同时支持非数据非唯一😅）

对标多个stl container的实现方式，估计要这么搞

```cpp
template<typename Iterator, typename Compare>
struct it_compare {
    bool operator()(const Iterator& x, const Iterator& y) const {
        return comp(*x, *y);
    }
private:
    Compare comp;
};//it_compare

typedef std::set<int> manual_t1;
typedef std::multiset<
    manual_t1::iterator,
    it_compare<
        manual_t1::iterator,
        std::greater<int>
    >
> manual_t2;
```

主要思路是一个容器`manual_t1`存储数据，另一个容器`manual_t2`则存储对应的迭代器，这个思路应该是正常我们实现的思路，每个容器都存储数据太浪费了。

数据的操作需要同时更新两个容器

```cpp
manual_t1 c1;
manual_t2 c2;

// insert the element 5
manual_t1::iterator it1=c1.insert(5).first;
c2.insert(it1);

// remove the element pointed to by it2
manual_t2::iterator it2=...;
c1.erase(*it2); // OK: constant time
c2.erase(it2);
```

[Spatial efficiency](http://www.boost.org/doc/libs/1_66_0/libs/multi_index/doc/performance.html#spatial_efficiency)一节从理论上分析了下，**multi_index_contaienr**更胜一筹。

[Time efficiency](http://www.boost.org/doc/libs/1_66_0/libs/multi_index/doc/performance.html#time_efficiency)一节开始pk时间复杂度。

测试的操作也很简单,n分别取1000,10000,100000

```cpp
multi_index_container<...> c;
for(int i=0;i<n;++i)c.insert(i);
for(iterator it=c.begin();it!=c.end();)c.erase(it++);
```

测试维度也比较广泛：
1. 1 ordered index，与`std::set` pk
2. 1 sequenced index，与`std::list` pk
3. 2 ordered indices，即前面说的数据结构
4. 1 ordered index + 1 sequenced index
5. 3 ordered indices(这里我感觉已经为了pk而pk了，也是为了证明索引越多优势越大的推论)
6. 2 ordered indices + 1 sequenced index

几轮测试结果，multi_index_container应该都是优于组合的stl containers的，直接贴下[官方结论](http://www.boost.org/doc/libs/1_66_0/libs/multi_index/doc/performance.html#conclusions)：

> We have shown that multi_index_container outperforms, both in space and time efficiency, equivalent data structures obtained from the manual combination of STL containers. This improvement gets larger when the number of indices increase.

>In the special case of replacing standard containers with single-indexed multi_index_containers, the performance of Boost.MultiIndex is comparable with that of the tested STL implementations, and can even yield some improvements both in space consumption and execution time.

但是我自己的测试结论显示部分情况下`boost::multi_index_container`要差一些，直接上我跑的结果吧

```
1 ordered index
  10^3 elmts: 123.93% (  0.14 ms /   0.11 ms)
  10^4 elmts: 121.01% (  1.83 ms /   1.52 ms)
  10^5 elmts: 107.88% ( 29.61 ms /  27.45 ms)
  space gain:  80.00%
1 sequenced index
  10^3 elmts:  92.16% (  0.04 ms /   0.04 ms)
  10^4 elmts:  91.84% (  0.36 ms /   0.40 ms)
  10^5 elmts:  90.89% (  3.63 ms /   3.99 ms)
  space gain: 100.00%
2 ordered indices
  10^3 elmts: 107.51% (  0.23 ms /   0.21 ms)
  10^4 elmts: 124.30% (  4.07 ms /   3.27 ms)
  10^5 elmts: 114.22% ( 58.00 ms /  50.78 ms)
  space gain:  70.00%
1 ordered index + 1 sequenced index
  10^3 elmts:  90.86% (  0.14 ms /   0.15 ms)
  10^4 elmts: 121.06% (  2.20 ms /   1.81 ms)
  10^5 elmts: 135.22% ( 33.73 ms /  24.94 ms)
  space gain:  75.00%
3 ordered indices
  10^3 elmts:  98.70% (  0.31 ms /   0.32 ms)
  10^4 elmts: 132.55% (  5.20 ms /   3.93 ms)
  10^5 elmts: 114.47% ( 81.15 ms /  70.89 ms)
  space gain:  66.67%
2 ordered indices + 1 sequenced index
  10^3 elmts:  91.52% (  0.23 ms /   0.25 ms)
  10^4 elmts:  70.26% (  3.10 ms /   4.41 ms)
  10^5 elmts:  91.34% ( 53.68 ms /  58.77 ms)
  space gain:  69.23%
```

百分比数值计算方式为：`multi_index_container`运行时间/组合的stl-container运行时间，可以看到有些情况下，相比还是要差一些的。

实际应用场景中，比如需要使用`std::set<int.`，我想很少有人会用`boost::multi_index_container<int>`代替吧，性能姑且不论，可读性还是stl大部分人更熟悉些，而且没准哪天boost里的代码就被放到了标准库里😎

最后必须要提的是[测试代码](http://www.boost.org/doc/libs/1_66_0/libs/multi_index/perf/test_perf.cpp)，这个封装与模板的使用，真的是大神级别orz，建议学习。

## 2. lru测试结果

官方结论总是需要方方面面，从理论分析，到多维度验证，考虑的太多。颇有些大厂各种数据跑分的架势。

我对比了之前见过的一个[lru cache](https://github.com/yingshin/Tiny-Tools/blob/master/cache/hash_map_list_lru.cpp)，对比上篇提到的实现的[lru cache implemented with multi_index_container](https://github.com/yingshin/Tiny-Tools/blob/master/cache/multi_index_lru.cpp)，测试结果

|N  |hash_map && list  |multi_index_container  |
|100000  |28181  |22758  |
|1000000  |270540  |181561  |
|10000000  |2.98237e+06  |2.07691e+06  |

表格内时间单位为us

可以看到是优于原来的lru cache实现的

## 3. 结论

boost官方文档说全面优于stl containers的组合，觉得未必可信。凡事还要理论 + 实践，从我的测试来看lru的实现上性能确实有提高，当然之前lru的实现应该本身有改进空间，欢迎指点。

从可读性上看，`multi_index_container`可读性要高很多：代码行数少、语义直接、不容易出错。
