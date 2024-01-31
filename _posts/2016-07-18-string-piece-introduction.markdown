---
title: StringPiece介绍
date: 2016-7-18 21:12:11
excerpt: "StringPiece介绍"
tags: cpp
---

记得最开始使用StringPiece的时候直接上手就用了，场景大概是这样子的：一个线程写StringPiece并放到队列里，另外一个线程从队列读取StringPiece，结果可想而知。后来发现很多项目源码里都会有这个类的类似实现，于是仔细看了下这个类的设计目的、实现。

本文是对阅读过程中的一些笔记的整理。

### 1. 为什么需要StringPiece

先看一个例子（摘自boost关于[string_ref的介绍](http://www.boost.org/doc/libs/1_61_0/libs/utility/doc/html/string_ref.html#string_ref.examples)）

```cpp
std::string extract_part ( const std::string &bar ) {
    return bar.substr ( 2, 3 );
    }

if ( extract_part ( "ABCDEFG" ).front() == 'C' ) { /* do something */ }
```

在上面代码执行过程中，即使经过RVO优化，仍旧不可避免的产生临时变量，例如首先"ABCDEFG"被转化成一个string作为形参，返回的substring，`front`产生的char等。

但是仔细分析下，这些临时变量不需要产生，例如传参时不需要拷贝，比如`const char*`，这样内存拷贝就可以避免。

这段代码说明了在传递`string`时，我们经常遇到的一个问题：

**不必要的拷贝**

在chrome的[StringPiece源码](https://cs.chromium.org/chromium/src/base/strings/string_piece.h?dr=CSs&q=string_piece.h&sq=package:chromium&l=1)里，说明了该类设计的主要目的：

> // You can use StringPiece as a function or method parameter.  A StringPiece  
> // parameter can receive a double-quoted string literal argument, a "const  
> // char*" argument, a string argument, or a StringPiece argument with no data  
> // copying.  Systematic use of StringPiece for arguments reduces data  
> // copies and strlen() calls.  

1. 统一了参数格式，不需要为`const char*` `const std::string&`等分别实现功能相同的函数了，参数统一指定为`StringPiece`即可  
2. 节约了数据拷贝以及`strlen`的调用  

根据boost里[string_ref](http://www.boost.org/doc/libs/1_61_0/libs/utility/doc/html/string_ref.html)的介绍，`StringPiece`同样适用于以下两种情况：  

1. 函数参数传入了`string`，而该函数内调用的另一个函数需要接收该string的一个substring  
2. 函数参数传入了`string`，而该函数需要return一个该string的substring  

boost里string_ref的实现跟`StringPiece`的想法是完全一致的，参考了[这篇文章](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2012/n3442.html)。

如何节省string的拷贝开销是一个非常普遍的需求，因此StringPiece的使用非常广泛，muduo引用了[pcre的StringPiece的实现](https://github.com/vmg/pcre/blob/master/pcre_stringpiece.h.in)，在llvm的源码里也有类似的[实现](http://llvm.org/docs/doxygen/html/StringRef_8h_source.html)。

### 2. chrome里的实现

以chrome里的StringPiece为例

`StringPiece`实际上是`BasicStringPiece`的特化，相关定义如下：

```cpp
template <typename STRING_TYPE> class BaseStringPiece;
typedef BasicStringPiece<std::string> StringPiece;
typedef BasicStringPiece<string16> StringPiece16;
```

其中`string16`封装的是`std::wstring wchar_t`

#### 2.1 构造及析构

根据上面提到的需求，BasicStringPiece的构造函数有多个，可以接收`const char*`，`const std::string&`：

```cpp
  BasicStringPiece() : ptr_(NULL), length_(0) {}
  BasicStringPiece(const value_type* str)
      : ptr_(str),
        length_((str == NULL) ? 0 : STRING_TYPE::traits_type::length(str)) {}
  BasicStringPiece(const STRING_TYPE& str)
      : ptr_(str.data()), length_(str.size()) {}
  BasicStringPiece(const value_type* offset, size_type len)
      : ptr_(offset), length_(len) {}
  BasicStringPiece(const typename STRING_TYPE::const_iterator& begin,
                    const typename STRING_TYPE::const_iterator& end)
      : ptr_((end > begin) ? &(*begin) : NULL),
        length_((end > begin) ? (size_type)(end - begin) : 0) {}
```

注意没有析构函数

同时成员变量只有两个:

```cpp
  const value_type* ptr_;
  size_type     length_;
```

因此

**BasicStringPiece没有字符串的控制权，不负责构造以及销毁。调用者需要保证在BasicStringPiece的生命周期里源buffer始终有效。**

#### 2.2 string操作

BasicStringPiece支持string的常见操作

```cpp
const value_type* data() const { return ptr_; }
void remove_prefix(size_type n);
void remove_suffix(size_type n);
void trim_spaces();
int compare(const BasicStringPiece<STRING_TYPE>& x) const;
STRING_TYPE as_string() const;

bool starts_with(const BasicStringPiece& x);
bool ends_with(const BasicStringPiece& x);
```
注意像`remove_prefix`这种操作都是常数级别的，因为只是在操作`ptr_`和`length_`

```cpp
  void remove_prefix(size_type n) {
    ptr_ += n;
    length_ -= n;
  }
```

行为上跟容器也很像，有自己的迭代器

```cpp
  value_type operator[](size_type i) const { return ptr_[i]; }
  const_iterator begin() const { return ptr_; }
  const_iterator end() const { return ptr_ + length_; }
  const_reverse_iterator rbegin() const {
    return const_reverse_iterator(ptr_ + length_);
  }
  const_reverse_iterator rend() const {
    return const_reverse_iterator(ptr_);
  } 
```

#### 2.3 字符串查找

同时支持find系列的需求

```cpp
size_type find(const BasicStringPiece<STRING_TYPE>& s,
    size_type pos = 0) const;
size_type rfind(const BasicStringPiece& s,
    size_type pos = BasicStringPiece::npos);
size_type find_first_of(const BasicStringPiece& s,
    size_type pos = 0) const;

template <typename STRING_TYPE>
const typename BasicStringPiece<STRING_TYPE>::size_type
BasicStringPiece<STRING_TYPE>::npos =
    typename BasicStringPiece<STRING_TYPE>::size_type(-1);
```

实现大部分也是通过`std::min search find find_end find_first_of`来完成。

其中像`find`实现里使用的是`std::search`

`find_first_of`并没有使用std里的`find_first_of`来实现，在参数`const BasicStringPiece& s`

+ 大小=1的情况下，退化为`find`查找  
+ 大小\>1的情况下，则建表查询，也就是以空间换时间的做法，  


```cpp
  bool lookup[UCHAR_MAX + 1] = { false };
  BuildLookupTable(s, lookup);
  for (size_t i = pos; i < self.size(); ++i) {
    if (lookup[static_cast<unsigned char>(self.data()[i])]) {
      return i;
    }
  }
```

BuildLookupTable则是遍历`s`建表

```cpp
// For each character in characters_wanted, sets the index corresponding
// to the ASCII code of that character to 1 in table.  This is used by
// the find_.*_of methods below to tell whether or not a character is in
// the lookup table in constant time.
// The argument `table' must be an array that is large enough to hold all
// the possible values of an unsigned char.  Thus it should be be declared
// as follows:
//   bool table[UCHAR_MAX + 1]
inline void BuildLookupTable(const StringPiece& characters_wanted,
                             bool* table) {
  const size_t length = characters_wanted.length();
  const char* const data = characters_wanted.data();
  for (size_t i = 0; i < length; ++i) {
    table[static_cast<unsigned char>(data[i])] = true;
  }
}
```

原因可能是stl的`find_first_of`的实现经常被简化为这样的伪代码：

```cpp
template<class ForwardIterator1, class ForwardIterator2>
  ForwardIterator1 find_first_of ( ForwardIterator1 first1, ForwardIterator1 last1,
                                   ForwardIterator2 first2, ForwardIterator2 last2)
{
  for ( ; first1 != last1; ++first1 )
    for (ForwardIterator2 it=first2; it!=last2; ++it)
      if (*it==*first1)          // or: if (comp(*it,*first)) for the pred version
        return first1;
  return last1;
}
```

对比下时间复杂度应该就能推测chrome修改的动机。但是一直没有系统的看过stl源码，因此不确定`std::find_first_of`是否可以简化为上面的伪代码。

对比了下boost的string_ref实现，boost多了`std::distance find_if`的实现，`find_first_of`也是直接使用的`std::find_first_of`。

顿时觉得chrome对代码的要求实在是精益求精。

#### 2.4 计算hash值

其实不太理解为什么字符串的hash值需要封装到StringPiece的接口里来

```cpp
std::size_t operator()(const base::StringPiece& sp) const;
```

### 3. 问题

muduo里大量使用了`const StringPiece& string_piece`，但是chrome里明确指出了prefer这种`void MyFunction(StringPiece arg);`使用方式，个人更赞同muduo的做法。
