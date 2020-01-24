---
title: "stl iterator介绍"
date: 2018-02-19 14:27:56
excerpt: "stl iterator"
tags: stl  iterator
---

最近整理文档，发现之前多看一个同事的分享，讲stl iterator的，讲的挺好，看了一遍后感觉现在用的确实少了。

因此有了这篇笔记，总结下stl里的迭代器。

<!--more-->

## 1. Why Iterator

我们知道stl里有很多算法，例如[find](http://en.cppreference.com/w/cpp/algorithm/find)，[sort](http://en.cppreference.com/w/cpp/algorithm/sort)，两者定义时模板名字不同

```
template< class InputIt, class T >
InputIt find( InputIt first, InputIt last, const T& value );

template< class RandomIt >
void sort( RandomIt first, RandomIt last );
```

虽然`InputIt RandomIt`只是一个模板名字，不过为什么不同？
甚至有的函数，例如[copy](http://en.cppreference.com/w/cpp/algorithm/copy)模板名字里有两种类型`InputIt OutputIt`

```
template< class InputIt, class OutputIt >
OutputIt copy( InputIt first, InputIt last, OutputIt d_first );
```

再看个例子：代码实现了接收用户输入的数字并输出到屏幕，遇到非数字则退出

```cpp
    std::copy(
            std::istream_iterator<int>(std::cin),
            std::istream_iterator<int>(),
            std::ostream_iterator<int>(std::cout, "\n"));
```

本文试图系统性的讲清楚这些之后的原理。

## 2. What Iterator

通常很多人对于stl的理解是两部分：

1. **container**，例如vector list set map...
2. **algorithm**，例如copy find sort...

而iterator则是作为两者之间的粘合剂

![containers_iterator_algorithm](/assets/images/containers_iterator_algorithm.png)

*注：图片摘抄自C++之父Bjarne Stroustrup的[Lecture slides ch20](http://www.stroustrup.com/Programming/lecture-slides.html)*

我们知道stl里container和其元素类型无关，这是通过模板技术做到的。

而通过iterator，则做到了algorithm和container无关，甚至我们可以把自定义iterator类型传入stl里的algorithm。

例如stl里的[find](http://en.cppreference.com/w/cpp/algorithm/find)，[copy](http://en.cppreference.com/w/cpp/algorithm/copy)参数都是iterator类型，而不是`const Container& c`。

确切的说，iterator是一种广义的指针

>Iterator is a generalized pointer identifying a position in a container.

![iterator](https://www.cs.helsinki.fi/u/tpkarkka/alglib/k06/lectures/iterator1.png)

同时像指针一样，支持`operator++ operator*`

>Allows moving in the container (operator++) and accessing elements (operator*)

## 3. Iterator Category

上面提到的`operator++ operator*`是一个iterator最基本的功能，我们考虑[find](http://en.cppreference.com/w/cpp/algorithm/find)的实现

```
template<class InputIt, class T>
InputIt find(InputIt first, InputIt last, const T& value)
{
    for (; first != last; ++first) {
        if (*first == value) {
            return first;
        }
    }
    return last;
}
```

可以看到find函数需要的iterator，还用到了`operator !=`。而根据iterator实现的接口不同，分为了5类.

![iterator category](/assets/images/iterator_category.png)

*注：ContiguousIterator在C++17引入，本文不做介绍*

可以看到所有的类型（除了`OutputIterator`）都继承自`InputIterator`，如果同时满足`OutputIterator`，那么就称为mutable的，否则称为`const_iterator`。

有些文档里会认为`ForwardIterator`继承自`InputIterator && OutputIterator`,

![iterator hierarchy](https://www.cs.helsinki.fi/u/tpkarkka/alglib/k06/lectures/iterator_concepts.png)

区别不大，相当于默认认为是可写的。

接下来，我们分别介绍下这五种iterator.

### 3.1. InputIterator

`InputIterator`里的input指的是`from container to the program`，即相对于program而言是input。

stl-algorithm里的`find`参数、`copy`里的源数据参数，都定义为`InputIterator`。

举个`istream_iterator`的例子：

```
    //构造std::cin对应的iterator
    std::istream_iterator<int> iter(std::cin);
    //读入iter的数据同时存储到value1 value2
    int value1 = *iter;
    int value2 = *iter;
    //输出value1 value2
    std::cout << value1 << std::endl;
    std::cout << value2 << std::endl;
```

注意没有调用`iter++`时，我们只能输入一个数字，value1=value2.

### 3.2. OutputIterator

`OutputIterator`的output，对应的，指的是`from program to the container`。

stl-algorithm里的`copy`里的目标参数类型，即为`OutputIterator`.

对应上面的例子，可以修改为

```
    std::istream_iterator<int> iter(std::cin);
    int value1 = *iter++;
    int value2 = *iter;
    std::ostream_iterator<int> iter_out(std::cout, "\n");
    //输出两个value
    *iter_out = value1;
    *iter_out = value2;
```

当然，使用`std::copy`，我们可以写出更加简洁的代码

```
    std::copy(
            std::istream_iterator<int>(std::cin),
            std::istream_iterator<int>(),
            std::ostream_iterator<int>(std::cout, "\n"));
```

### 3.3. ForwardIterator

`ForwardIterator`增加了`mulit-pass`。

这里我们先引入一个`one-pass` `multi-pass`的概念，其中`InputIterator/OutputIterator`都是`one-pass`的，即任何情况下都只有一个active的iterator，我们看个`istream_iterator`的例子:

```
    // 定义istream_iterator，输入1
    auto iter1 = std::istream_iterator<int>(std::cin);
    auto iter2 = iter1;
    // *iter1:1
    std::cout << "*iter1:" << *iter1 << std::endl;
    // *iter2:1
    std::cout << "*iter2:" << *iter2 << std::endl;
    // increment iter1，输入2
    ++iter1;
    // increment iter2，输入3
    ++iter2;
    // *iter1:2
    std::cout << "*iter1:" << *iter1 << std::endl;
    // *iter2:3
    std::cout << "*iter2:" << *iter2 << std::endl;
```

可以看到虽然iter1 iter2初始相等，但是分别++后指向了不同的value。

也就是说，对于`std::cin`，只能有一个iterator允许自增操作，这个iterator称为active的。或者是`iter1`，或者是`iter2`，指向新输入的value，而不能有两个active的iterator，都自增，然后指向同一个value。即

>input iterator x == y does not imply ++x == ++y.  forward iterator x == y implies ++x == ++y.

更多区别可以参考[iterator](http://www.cplusplus.com/reference/iterator/)。

stl-algorithm里的[replace](http://en.cppreference.com/w/cpp/algorithm/replace)，[fill](http://en.cppreference.com/w/cpp/algorithm/fill), [adjacent_find](http://en.cppreference.com/w/cpp/algorithm/adjacent_find)参数都需要的是`ForwardIterator`

```
template< class ForwardIt, class T >
void replace( ForwardIt first, ForwardIt last,
              const T& old_value, const T& new_value );

template< class ForwardIt, class T >
void fill( ForwardIt first, ForwardIt last, const T& value );

template< class ForwardIt >
ForwardIt adjacent_find( ForwardIt first, ForwardIt last );
```

考虑下`adjacent_find`的实现

```
template<class ForwardIt>
ForwardIt adjacent_find(ForwardIt first, ForwardIt last)
{
    if (first == last) {
        return last;
    }
    ForwardIt next = first;
    ++next;
    for (; next != last; ++next, ++first) {
        if (*first == *next) {
            return first;
        }
    }
    return last;
}
```

注意`++next, ++first`这句，因此算法需要iterator支持`multi-pass operator++`

我们看一个`forward_list`的例子

```
    std::forward_list<int> flist{1, 2, 5, 4, 3};
    auto iter = flist.begin();
    //2
    std::cout << *++iter << std::endl;
    // compile error: no match for 'operator--' (operand type is 'std::_Fwd_list_iterator<int>')
    //std::cout << *--iter << std::endl;
    // compile error: no match for 'operator+' (operand types are 'std::_Fwd_list_iterator<int>' and 'int' )
    // std::cout << *(iter + 1)<< std::endl;
```

`*++iter`可以正确获取对应的元素。

`*--iter *(iter + 1)`则会导致compile error，因为`std::forward_list::iterator`没有实现`operator-- operator+`

### 3.4. BidirectionalIterator

`BidrectionalIterator`增加了`operator--`

stl-algorithm里的[reverse](http://en.cppreference.com/w/cpp/algorithm/reverse),[inplace_merge](http://en.cppreference.com/w/cpp/algorithm/inplace_merge)都需要传入`BidirectionalIterator`。

```
template< class BidirIt >
void reverse( BidirIt first, BidirIt last );

template< class BidirIt >
void inplace_merge( BidirIt first, BidirIt middle, BidirIt last );
```

考虑`reverse`的pseudocode

```
template<class BidirIt>
void reverse(BidirIt first, BidirIt last)
{
    while ((first != last) && (first != --last)) {
        std::iter_swap(first++, last);
    }
}
```

可以看到`--last`用到了`operator--`，也就是算法需要的iterator需要支持`opertor--`

我们把`ForwardIterator`例子里的`forward_list`换成`list`

```
    std::list<int> list{1, 2, 5, 4, 3};
    auto iter = list.begin();
    //2
    std::cout << *++iter << std::endl;
    //1
    std::cout << *--iter << std::endl;
    // compile error: no match for 'operator+' (operand types are 'std::_List_iterator<int>' and 'int')
    // std::cout << *(iter + 1)<< std::endl;
```

`*++iter *--iter`可以正确获取对应的元素，`*(iter + 1)`仍然会导致编译错误（提示略有不同）。

### 3.5. RandomAccessIterator

`RandomAccessIterator`增加了随机访问的功能

stl-algorithm里的[sort](http://en.cppreference.com/w/cpp/algorithm/sort)需要传入`RandomAccessIterator`。

```
template< class RandomIt >
void sort( RandomIt first, RandomIt last );
```

因为`sort`实现时用到了快排、堆排序和插入排序，需要用到iterator的`operator- operator+`(例如快排分为两部分时枢纽位置的pivot-1 pivot+1)。

`vector/deque`的iterator满足这个需求，我们继续更新`BidirectionalIterator`的例子，`list`换成`vector`

```
    std::vector<int> vec{1, 2, 5, 4, 3};
    auto iter = vec.begin();
    //2
    std::cout << *++iter << std::endl;
    //1
    std::cout << *--iter << std::endl;
    //2
    std::cout << *(iter + 1)<< std::endl;
```

可以看到`opertor++ -- - +`都可以正确获取对应的元素。

## 3. Required vs Provided

stl-algorithm里的算法有分别需要的最低级别的iteator，例如`find`需要`InputIterator`，而`sort`需要的是`RandomIterator`，我们称为`iterator required`。

而对于stl-containers，则提供了对应的iterator，称为`iterator provided`。

前面我们也有介绍到，总结下：

|Iterator Category  |Ability  |Providers  |
|--|--|--|
|Output iterator  |Writes forward  |Ostream, inserter  |
|Input iterator  |Reads forward once  |Istream  |
|Forward iterator  |Reads forward  |Forward list, unordered containers  |
|Bidirectional iterator  |Reads forward and backward  |List, set, multiset, map, multimap  |
|Random-access iterator  |Reads with random access  |Array, vector, deque, string, C-style array  |

*注：上表摘抄自The C++ Standard Library Second Edition*

可以看到C-style array属于RandomAccessIteratory，属于最高级别的iteator，因此能够用于stl-algorithm.

## 4. Iterator Helper Function

stl里提供了一些函数用于处理iterator。

前面介绍了不同iterator接口不同，比如我们有个需求，对当前iterator指向往后+5个位置

如果是`RandomAccessIteator`，我们可以直接操作

```
iter += 5;
```

但是对于`Non RandomAccessIterator`，我们需要一个for-loop

```
for (int i = 0; i < 5; ++i)
  iter++;
```

stl提供了一些help function，封装了这些对于不同类型迭代器的实现，使用起来更加优雅一些。

### 4.1. advance

```
template< class InputIt, class Distance >
void advance( InputIt& it, Distance n );
```

`advance`修改输入的`it`，n可以取正负数。

注意需要调用方自己保证：
1. iterator在合理range内，例如[c.begin, c.end)
2. 如果n为负数，需要iterator支持backward

否则行为是不确定的

```
    std::forward_list<int> c{1, 2, 3, 4, 5};
    auto iter = c.begin();
    //1
    std::cout << *iter << std::endl;
    std::advance(iter, 2);
    //3
    std::cout << *iter << std::endl;
    // std::advance(iter, 1);
    // std::advance(iter, -1);
```

注意最后两行，在我的测试环境下会导致程序core掉，在有些编译器上会无法编译。

[cppreference](http://en.cppreference.com/w/cpp/iterator/advance)上说明了**undefined behavior**的情况：

>If n is negative, the iterator is decremented. In this case, InputIt must meet the requirements of BidirectionalIterator, otherwise the behavior is undefined.

>The behavior is undefined if the specified sequence of increments or decrements would require that a non-incrementable iterator (such as the past-the-end iterator) is incremented, or that a non-decrementable iterator (such as the front iterator or the singular iterator) is decremented.

类似的还有类似于我们要构造一个 string 的倒置：

```
    std::string A = "abc";
    std::string B;

    // std::reverse_copy(A.begin(), A.end(), B.begin());//undefined behavior
    std::reverse_copy(A.begin(), A.end(), std::back_inserter(B));//right
```

时间复杂度上是O(n)，如果是`RandomAccessIterator`，则是O(1)。

`advance`经常用来跳过`std::cin`的输入，例如这样忽略前两个输入的`int`值：

```
    std::istream_iterator<int> iter(std::cin);
    advance(iter, 2);
```

### 4.2. next/prev

`next/prev`在c++11引入，与`advance`最大的不同是不修改输入参数，而是通过返回值的形式。

```
template< class ForwardIt >
ForwardIt next(
  ForwardIt it, 
  typename std::iterator_traits<ForwardIt>::difference_type n = 1 );

template< class BidirIt >
BidirIt prev(
  BidirIt it, 
  typename std::iterator_traits<BidirIt>::difference_type n = 1 );
```

关于性能上，`The C++ Standard Library Second Edition`在`next`里提到了

>Calls advance(pos,n) for an internal temporary object.

没有理解，我理解的是`next`会引入额外的临时变量，了解的朋友欢迎交流下。（感觉抠性能抠到这个份上一定是个大神了🙂）

从官方文档理解，应该是推荐用`next/prev`。

### 4.3. distance

`distance`用于计算两个`iterator`的距离

```
template< class InputIt >
typename std::iterator_traits<InputIt>::difference_type 
    distance( InputIt first, InputIt last );
```

### 4.4. iter_swap

```
template< class ForwardIt1, class ForwardIt2 >
void iter_swap( ForwardIt1 a, ForwardIt2 b );
```

注意从声明上可以看到iter_swap的两个参数不需要相同类型，只要对应的value能够互相转化即可

```
    std::vector<int> v{1, 2, 3, 4, 5};
    std::list<int> l{6, 7, 8, 9, 10};

    std::iter_swap(v.begin(), next(v.begin()));
    std::iter_swap(v.begin(), l.begin());

    //6
    //1
    //3
    //4
    //5
    std::copy(
            v.begin(),
            v.end(),
            std::ostream_iterator<int>(std::cout, "\n"));
    //2
    //7
    //8
    //9
    //10
    std::copy(
            l.begin(),
            l.end(),
            std::ostream_iterator<int>(std::cout, "\n"));
```

## 5. Iterator Adaptors

`Input/Output/Forward/Bidirectional/RandomAccess`这些都只是定义了`iterator`的类别。

第三节里我们介绍了stl-containers里provide的`iterator`分别满足哪种类别。

为了方便stl-algorithms里的各个算法，stl还预定义了一些具体化的`iterator`，它们定义了具体行为，对算法起到了极大的辅助作用。

### 5.1. Reverse Iterator

Reverse Iterator提供了反方向遍历的能力，与iterator的位置对应如图

![Reverse Iterator](https://www.cs.helsinki.fi/u/tpkarkka/alglib/k06/lectures/reverse_iterator.png)

可以看到对于iterator:

+ begin指向第一个元素
+ end指向最后一个元素下一个位置，也就是past the end

对于reverse iterator则正好相反：

+ rbegin指向最后一个元素
+ rend指向第一个元素上一个位置

`iterator`与`reverse_iterator`可以互相转换，其中`iter`指向对应的`reverse_iterator`下一个位置。

看个例子

```
    std::vector<int> v{1, 2, 3, 4, 5};

    //iter指向第4个元素，即value = 4
    std::vector<int>::iterator iter = v.begin() + 3;
    //riter由iter得到，指向iter前一个元素
    std::vector<int>::reverse_iterator riter(iter);
    //也可以直接用std::reverse_iterator定义
    // std::reverse_iterator<std::vector<int>::iterator> riter(iter);
    std::cout << *iter << std::endl;//4
    //iter +1向后一个位置
    std::cout << *(iter + 1) << std::endl;//5
    std::cout << *riter << std::endl;//3
    //riter +1向前一个位置
    std::cout << *(riter + 1) << std::endl;//2
    //使用base获取iter位置，两者相同
    std::cout << *iter.base() << std::endl;//4
    std::cout << *riter.base() << std::endl;//4
```


### 5.2. Insert Iterator

Insert Iterator分为三种

1. Back Insert
2. Front Insert
3. General Insert

例如我们希望将输入数据存储到vector

```
    std::vector<int> v;
    std::copy(
            std::istream_iterator<int>(std::cin),
            std::istream_iterator<int>(),
            v.begin());
```

这么写会导致**undefined behavior**，因为v并没有足够的空间存储输入数据，reverse保证空间是一个办法。更优雅的办法是使用`insert`

```
    std::vector<int> v;
    std::copy(
            std::istream_iterator<int>(std::cin),
            std::istream_iterator<int>(),
            // std::back_inserter(v));
            std::inserter(v, v.end()));
```

注意使用`std::inserter std::back_inserter`都是可以的，不过使用`front_inserter`则会导致编译错误

```
    std::vector<int> v;
    std::copy(
            std::istream_iterator<int>(std::cin),
            std::istream_iterator<int>(),
            //error: ‘class std::vector<int>’ has no member named ‘push_front’
            std::front_inserter(v));
```

因为`std::front_inserter`实现上调用的是`push_front`，而`std::vector`并没有这个接口。

各种inserter对应的function列表如下：

|Name  |Class  |Called Function  |Creation  |
|--|--|--|--|
|Back inserter  |back_insert_iterator  |push_back(value)   |back_inserter(cont)  |
|Front inserter  |front_insert_iterator  |push_front(value)  |front_inserter(cont)  |
|General inserter  |insert_iterator  |insert(pos,value)  |inserter(cont,pos)  |

### 5.3. Stream Iterator

Stream Iterator包含两种

1. istream_iterator
2. ostream_iterator

前面出现过很多次，不多做介绍了

## 6.Iterator Traits

Traits的思想在stl里几乎处处都是，我们再回顾下。

### 6.1. Traits

例如我们实现`iter_swap`这个函数，需要使用`value_type`指定数据类型

```
template <class Iterator>
void iter_swap (Iterator a, Iterator b) {
  // define value_type
  value_type tmp = *a;
  *a = *b;
  *b = tmp;
}
```

一个想法是为每种`iterator`都定义一个`value_type`。（或许用C++11里的`decltype`也可以？）

```
struct my_iterator {
  typedef ... value_type;
  ...
};
```

然而这个想法对指针却不适用，因为指针不是一个struct。

stl里的解决方法就是定义一个辅助模板类

```
template <class Iterator>
struct iterator_traits {
  typedef typename iterator::value_type value_type;
};
```

对指针则偏特化

```
template <class T>
struct iterator_traits<T*> {
  typedef T value_type;
};
```

这样我们就可以在`iter_swap`里这么使用

```
typedef typename iterator_traits<Iterator>::value_type value_type;
value_type ...;
```

`std::iterator_traits`定义了更多类型（这也是decltype不是万金油的原因）

```
template < class Iterator > struct iterator_traits {
  typedef typename Iterator::value_type        value_type ;
  typedef typename Iterator::difference_type   difference_type ;
  typedef typename Iterator::pointer           pointer ;
  typedef typename Iterator::reference         reference ;
  typedef typename Iterator::iterator_category iterator_category ;
};

//指针类型的特化
template < class T > struct iterator_traits <T* > {
  typedef T                          value_type ;
  typedef ptrdiff_t                  difference_type ;
  typedef T*                         pointer ;
  typedef T&                         reference ;
  typedef random_access_iterator_tag iterator_category ;
};

template < class T > struct iterator_traits <const T* > {
  typedef T                          value_type ;
  typedef ptrdiff_t                  difference_type ;
  typedef const T*                   pointer ;
  typedef const T&                   reference ;
  typedef random_access_iterator_tag iterator_category ;
};
```

整理成表格即：

|Member type  |Definition  |
|--|--|
|difference_type  |Iterator::difference_type  |
|value_type  |Iterator::value_type  |
|pointer  |Iterator::pointer  |
|reference  |Iterator::reference  |
|iterator_category  |Iterator::iterator_category  |

对指针类型也有对应的特化

|Member  |type Definition  |
|--|--|
|difference_type  |std::ptrdiff_t  |
|value_type  |T (until C++20)std::remove_cv_t<T> (since C++20)  |
|pointer  |T*  |
|reference  |T&  |
|iterator_category  |std::random_access_iterator_tag  |

具体可以参考<http://en.cppreference.com/w/cpp/iterator/iterator_traits>

### 6.2. Tag Dispatching

再考虑`advance`的实现

```
template< class InputIt, class Distance >
void advance( InputIt& it, Distance n );
```

我们知道`advance`的时间复杂度跟iterator类型有关（是否支持+=）

```
Linear.
However, if InputIt additionally meets the requirements of RandomAccessIterator, complexity is constant.
```

即对于`RandomAccessIterator`，实现可能是这样子的

```
template <class RandomAccessIterator, class Distance>
void advance (RandomAccessIterator& i, Distance n) {
  i += n;
}
```

stl的解决方法是定义tag

```
struct input_iterator_tag { };
struct output_iterator_tag { };
struct forward_iterator_tag : public input_iterator_tag { };
struct bidirectional_iterator_tag : public forward_iterator_tag { };
struct random_access_iterator_tag : public bidirectional_iterator_tag { };
```

5种tag分别对应5种类型的`iterator`。

`std::iterator`是一个模板类，通过传入不同的`iterator_category`标记不同的`iterator`类型，具体可以参考[iterator_traits](http://en.cppreference.com/w/cpp/iterator/iterator_tags)，原理即

```
foo(Iterator Iter)
->
foo(Iterator Iter, typename std::iterator_traits<Iter>::iterator_category())
->
foo(Iterator Iter, std::input_iterator_tag)
...
foo(Iterator Iter, std::random_access_iterator_tag)
```

当然，借助[std::enable_if](http://izualzhy.cn/SFINAE-and-enable_if)我们可以有另一种实现方案（原理相同）

```
template<class BDIter,
        typename std::enable_if<
            std::is_base_of<std::bidirectional_iterator_tag,
            typename std::iterator_traits<BDIter>::iterator_category >::value &&
            !std::is_same<std::random_access_iterator_tag,
            typename std::iterator_traits<BDIter>::iterator_category >::value
        >::type* = nullptr
    >
void alg(BDIter, BDIter)
{
    std::cout << "alg() called for bidirectional iterator\n";
}

template<class BDIter,
        typename std::enable_if<
            std::is_base_of<std::random_access_iterator_tag,
            typename std::iterator_traits<BDIter>::iterator_category >::value
        >::type* = nullptr
    >
void alg(BDIter, BDIter)
{
    std::cout << "alg() called for random-access iterator\n";
}
```

### 6.3 User-Define Iterator

自定义iterator最好的办法是继承stl里的`iterator`

```
template < class Category , class Value , class Distance = ptrdiff_t ,
           class Pointer = Value*, class Reference = Value&>
struct iterator {
  typedef Category  category ;
  typedef Value     value_type ;
  typedef Distance  difference_type ;
  typedef Pointer   pointer ;
  typedef Reference reference ;
}
```

例如我们可以这么定义`my_iterator`，并应用stl里的`find`算法。

```
struct node {
  int val;
  node* next;

  node (int i, node* n) : val(i), next(n) {}
  // implicit copy constructor, copy assignment and destructor
  // no default constructor

  struct my_iterator
      : std::forward_list<int>::iterator
    // : std::iterator<std::forward_iterator_tag, int>
  {
    node* ptr;

    explicit my_iterator (node* p = 0) : ptr(p) {}
    // implicit copy constructor, copy assignment and destructor

    reference operator* () { return ptr->val; }

    my_iterator& operator++ () { ptr = ptr->next; return *this; }
    my_iterator operator++ (int) {
      my_iterator tmp = *this; ++*this; return tmp; }

    bool operator== (const my_iterator& other) const {
      return ptr == other.ptr;
    }
    bool operator!= (const my_iterator& other) const {
      return ptr != other.ptr;
    }
  };

  struct const_iterator
      : std::forward_list<int>::const_iterator
    // : std::iterator<std::forward_iterator_tag, int, std::ptrdiff_t,
               // const int*, const int&>
  {
    const node* ptr;

    explicit const_iterator (const node* p = 0) : ptr(p) {}
    // implicit copy constructor, copy assignment and destructor
    const_iterator (const my_iterator& i) : ptr(i.ptr) {}

    reference operator* () { return ptr->val; }

    const_iterator& operator++ () { ptr = ptr->next; return *this; }
    const_iterator operator++ (int) {
      const_iterator tmp = *this; ++*this; return tmp; }

    bool operator== (const const_iterator& other) const {
      return ptr == other.ptr;
    }
    bool operator!= (const const_iterator& other) const {
      return ptr != other.ptr;
    }
  };

  friend
  bool operator== (my_iterator a, const_iterator b) {
    return a.ptr == b.ptr;
  }
  friend
  bool operator!= (my_iterator a, const_iterator b) {
    return a.ptr != b.ptr;
  }
  friend
  bool operator== (const_iterator a, my_iterator b) {
    return a.ptr == b.ptr;
  }
  friend
  bool operator!= (const_iterator a, my_iterator b) {
    return a.ptr != b.ptr;
  }

  my_iterator begin() { return my_iterator(this); }
  my_iterator end()   { return my_iterator(); }

  const_iterator begin() const { return const_iterator(this); }
  const_iterator end()   const { return const_iterator(); }

};

int main() {
    node* list = new node(3,0);
    list = new node(2, list);
    list = new node(1, list);
    node::my_iterator iter = std::find(list->begin(), list->end(), 2);
    assert (*iter==2);
    return 0;
}
```

# 7. Conclusion

stl-iterator实际上是一个很庞大的话题，细细展开已经不是一篇文章两万字可以讲清楚的了，例如iterator的设计思想，tag dispatcher，traits的设计，包括half-open的range:[begin, end)已经成为我们对于range一个标准的理解。而stl还在不断的完善这个feature，例如`Move Iterators`的设计。伴随stl-algorithm function，iterator可以呈现更大的空间，之后会写一篇文章总结下stl里的algorithm。

当然iterator的使用也有很多需要注意的地方：**singular past-the-end out-of-range dangling inconsistent**等，如果有时间也想总结一篇。

18年正式到来了，感觉有很多技术点想要分享🐶。

熟练使用iterator，离我们对于代码的追求能更进一步：
**more effecient, less buggy, more reable, more clean**

# 8. 参考资料

1. [helsinki公开课](https://www.cs.helsinki.fi/u/tpkarkka/alglib/k06/lectures/iterators.html)
