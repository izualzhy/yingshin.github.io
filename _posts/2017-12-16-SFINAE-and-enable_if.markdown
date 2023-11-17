---
title: "C++模板技术之SFINAE与enable_if的使用"
date: 2017-12-16 10:36:23
excerpt: "C++模板技术之SFINAE与enable_if的使用"
tags: cpp
---

想写这篇文章主要是偶然看到很多代码使用[protobuf里的反射](http://izualzhy.cn/protobuf-message-reflection)，例如我们想要获取字段`common.logid`对应的value，很多实现使用`GetReflection/GetString/field/HasFeild`这些接口来获取。

```cpp
/*
部分schema定义
message Common {
    optional string logid = 1;
}
optional Common common = 1;
*/

    const google::protobuf::Descriptor* descriptor = message->GetDescriptor();
    const google::protobuf::Reflection* reflection = message->GetReflection();
    const google::protobuf::FieldDescriptor* common_field =
        descriptor->FindFieldByName("common");
    assert(common_field != NULL);

    const google::protobuf::Message& common_message = reflection->GetMessage(*message, common_field);
    const google::protobuf::Descriptor* common_descriptor = common_message.GetDescriptor();
    const google::protobuf::Reflection* common_reflection = common_message.GetReflection();
    const google::protobuf::FieldDescriptor* logid_field =
        common_descriptor->FindFieldByName("logid");
    assert(logid_field != NULL);
    return common_reflection->GetString(common_message, logid_field);
```

虽然pb里的反射实现上性能并没有明显降低，不过一直在思考能否有更简洁的方案实现，比如直接调用`logid`这个方法，但是不知道对象类型的情况下，这么写一定会导致编译错误。

而解决的方案就在于SFINAE.

<!--more-->

## 1. SFINAE是什么

SFINAE是C++模板编译的一个原则，全名叫做**Substitution Failure Is Not An Error**

C++编译过程中，如果模板实例化后的某个模板函数（模板类）对该调用无效，那么将继续寻找其他重载决议，而不是引发一个编译错误。

> To summarize, the essence of the SFINAE principle is this: If an invalid argument or return type is formed when a function template is instantiated during overload resolution, the function template instantiation is removed from the overload resolution set and does not result in a compilation error.

说着比较绕口，实际编程中我们或多或少都使用过这个原则，看个例子：

```cpp
struct Test {
    typedef int foo;
};

template <typename T>
//要求类型T定义了内嵌类型foo
void f(typename T::foo) {} // Definition #1

template <typename T>
void f(T) {}               // Definition #2

int main() {
    f<Test>(10); // Call #1.
    f<int>(10);  // Call #2. Without error (even though there is no int::foo) thanks to SFINAE.

    return 0;
}
```

模板函数`f`一共定义了两个版本。
`f<int>`在适配version-1时因为没有`nested type: foo`，会引发一个`template argument deduction/substitution failed`，编译器不会报错，而是通过version-2实例化出一个`f<int>`来。

nm可以看到bin文件里的符号名

```
00000000004007d1 W void f<Test>(Test::foo)
00000000004007da W void f<int>(int)
```

## 2. SFINAE的例子

SFINAE原则最开始被设计出来，是应用于上面C++模板实例化的编译场景。

而后来C++  developers发现SFINAE可以用于编译期决断，特别是配合sizeof使用下，可以方便的判断诸如：类是否定义了内嵌类型，是否定义了给定名字的成员函数等。

### 2.1. 查看类是否定义了内嵌类型

例如我们想查看类T是否有`T::iterator`这个类型

```cpp
#include <iostream>
#include <vector>

template <typename T>
struct has_typedef_iterator {
    // Types "yes" and "no" are guaranteed to have different sizes,
    // specifically sizeof(yes) == 1 and sizeof(no) == 2.
    typedef char yes[1];
    typedef char no[2];

    template <typename C>
    static yes& test(typename C::iterator*);

    template <typename>
    static no& test(...);

    // If the "sizeof" of the result of calling test<T>(nullptr) is equal to sizeof(yes),
    // the first overload worked and T has a nested type named iterator.
    static const bool value = sizeof(test<T>(nullptr)) == sizeof(yes);
};

struct foo {
    typedef float iterator;
};

int main() {
    std::cout << std::boolalpha;
    std::cout << has_typedef_iterator<foo>::value << std::endl;//true
    std::cout << has_typedef_iterator<int>::value << std::endl;//false
    std::cout << has_typedef_iterator<std::vector<int> >::value << std::endl;//true

    return 0;
}
```

如果T定义了内嵌类型(nested type): iterator，例如`foo,std::vector<...>`，那么会适配第一个`test`模板函数，通过传入`nullptr`我们可以省略了构造一个对象指针的过程，返回值为`yes`。

如果T未定义iterator，例如int，由于SFINAE原则，适配第一个失败后编译器继续适配第二个并且成功，返回值为`no`。

由于`yes no`被特意设计了不同的size，配合sizeof，我们就可以判断传入的类型是否定义了`iterator`。

注意编译器有一个最优原则，比如`foo`同样适配于第二个`test`模板，但是第一个`test`优先级更高(更加适配)，因此编译器选择最优的第一个`test`模板，并不会报`ambiguous`。

### 2.2. 现代C++查看类是否定义了内嵌类型

上面的功能，C++11的语法实现上要更简洁一些

```cpp
#include <iostream>
#include <vector>
#include <type_traits>

template <typename U>
struct iterator_help {
    typedef void iterator;
};

template <typename T, typename = void>
struct has_typedef_iterator : std::false_type {};

template <typename T>
struct has_typedef_iterator<T, typename iterator_help<typename T::iterator>::iterator > : std::true_type {};

struct foo {
    typedef float iterator;
};

int main() {
    std::cout << std::boolalpha;
    std::cout << has_typedef_iterator<foo>::value << std::endl;//true
    std::cout << has_typedef_iterator<int>::value << std::endl;//false
    std::cout << has_typedef_iterator<std::vector<int> >::value << std::endl;//true

    return 0;
}
```

还有一些更加简洁的写法，需要更高级版本的编译器支持，具体可以参考附录里wiki的内容。

### 2.3. 使用boost查看类是否定义了内嵌类型

boost.mpl提供了BOOST_MPL_HAS_XXX_TRAIT_DEF，可以协助我们方便的测试类型T是否定义了某个内嵌类型。

```cpp
#include <iostream>
#include <vector>
#include "boost/mpl/has_xxx.hpp"

BOOST_MPL_HAS_XXX_TRAIT_DEF(iterator);

int main() {
    std::cout << std::boolalpha;
    std::cout << has_iterator<int>::value << std::endl;//false
    std::cout << has_iterator<std::vector<int> >::value << std::endl;//true

    return 0;
}
```

### 2.4 查看类是否定义了某个成员函数

[An introduction to C++'s SFINAE concept: compile-time introspection of a class member](https://jguegant.github.io/blogs/tech/sfinae-introduction.html)详细介绍了从c98到c++17的做法

这里介绍下boost里tti的用法   。

```cpp
#include <vector>
#include <iostream>
#include "boost/tti/has_member_function.hpp"

BOOST_TTI_HAS_MEMBER_FUNCTION(foo);
BOOST_TTI_HAS_MEMBER_FUNCTION(bar);

struct Foo {
    void foo();
};//foo

struct Bar {
    void bar();
};//Bar

int main() {
    std::cout << std::boolalpha
        << has_member_function_foo<Foo, void>::value << std::endl
        << has_member_function_bar<Bar, void>::value << std::endl;

    return 0;
}
```

## 3. enable_if的作用

`enable_if`的出现使得SFINAE使用上更加方便，进一步扩展了上面`has_xxx is_xxx`的作用。

而`enable_if`实现上也是使用了SFINAE。

boost与std里都有定义，接下来的例子可能都有用到。

### 3.1. cppreference的例子

首先从cppreference的例子看下enable_if的两种用法

```cpp
// enable_if example: two ways of using enable_if
#include <iostream>
#include <type_traits>

// 1. the return type (bool) is only valid if T is an integral type:
template <class T>
typename std::enable_if<std::is_integral<T>::value,bool>::type
  is_odd (T i) {return bool(i%2);}

// 2. the second template argument is only valid if T is an integral type:
template < class T,
           class = typename std::enable_if<std::is_integral<T>::value>::type>
bool is_even (T i) {return !bool(i%2);}

int main() {

  short int i = 1;    // code does not compile if type of i is not integral

  std::cout << std::boolalpha;
  std::cout << "i is odd: " << is_odd(i) << std::endl;
  std::cout << "i is even: " << is_even(i) << std::endl;

  return 0;
}
```

上面代表了enable_if的两种惯用方法：

1. 返回值类型使用enable_if  
2. 模板参数额外指定一个默认的参数class = typename std::enable_if<...>::type  

推荐使用第一种，更方便些。

使用enable_if的好处是控制函数只接受某些类型的(value==true)的参数，否则编译报错。

比如如果我们增加这么一句

```cpp
std::cout << "i is even: " << is_even(100.0) << std::endl;
```

编译失败提示：

```
test_enable_if.cpp: In function ‘int main()’:
test_enable_if.cpp:37:46: error: no matching function for call to ‘is_even(double)’
   std::cout << "i is even: " << is_even(100.0) << std::endl;
                                              ^
test_enable_if.cpp:37:46: note: candidate is:
test_enable_if.cpp:28:6: note: template<class T, class> bool is_even(T)
 bool is_even (T i) {return !bool(i%2);}
      ^
test_enable_if.cpp:28:6: note:   template argument deduction/substitution failed:
test_enable_if.cpp:27:12: error: no type named ‘type’ in ‘struct std::enable_if<false, void>’
            class = typename std::enable_if<std::is_integral<T>::value>::type>
```

注意这么一句是关键：

**error: no type named ‘type’ in ‘struct std::enable_if<false, void>’**

接下来我们看下`enable_if`的实现，就能找到这句编译失败提示的来源。

### 3.2. enable_if的实现

C++模板元编程里的定义

```cpp
template <bool, typename T=void>
struct enable_if {
};

template <typename T>
struct enable_if<true, T> {
  using type = T;
};
```

可以看到当enable_if第一个类型为true时会特化到第二种实现，此时内嵌类型type存在。

否则编译器匹配第一种实现，内嵌类型type不存在，这也是上面编译操作提示的原因。

boost里的实现位于core/enable_if.cpp

```cpp
namespace boost
{

  template <bool B, class T = void>
  struct enable_if_c {
    typedef T type;
  };

  template <class T>
  //false时不存在内嵌类型type
  struct enable_if_c<false, T> {};

  template <class Cond, class T = void>
  //::value是type_traits的常用静态变量定义
  struct enable_if : public enable_if_c<Cond::value, T> {};

  template <bool B, class T>
  struct lazy_enable_if_c {
    typedef typename T::type type;
  };

  template <class T>
  struct lazy_enable_if_c<false, T> {};

  template <class Cond, class T>
  struct lazy_enable_if : public lazy_enable_if_c<Cond::value, T> {};

  //disable与enbale相反,true时不存在type类型
  template <bool B, class T = void>
  struct disable_if_c {
    typedef T type;
  };

  template <class T>
  struct disable_if_c<true, T> {};

  template <class Cond, class T = void>
  struct disable_if : public disable_if_c<Cond::value, T> {};

  template <bool B, class T>
  struct lazy_disable_if_c {
    typedef typename T::type type;
  };

  template <class T>
  struct lazy_disable_if_c<true, T> {};

  template <class Cond, class T>
  struct lazy_disable_if : public lazy_disable_if_c<Cond::value, T> {};

} // namespace boost
```

### 3.3. enable_if的使用

借助于boost的众多type_traits接口，可以很方便的决定：

将/不将某种类型加入到我们模板实例化决议集合里。

```cpp
#include <boost/utility/enable_if.hpp>
#include <type_traits>
#include <string>
#include <iostream>

template <typename T>
void print(typename boost::enable_if<std::is_integral<T>, T>::type i) {
    std::cout << "Integral: " << i << '\n';
}

template <typename T>
void print(typename boost::enable_if<std::is_floating_point<T>, T>::type f) {
    std::cout << "Floating point: " << f << '\n';
}

int main()
{
    print<short>(1);
    print<long>(2);
    print<double>(3.14);

    return 0;
}
```

boost里type_traits还有例如`is_class has_trival_copy has_virtual_detructor`等，具体可以参考boost官方文档[type_traits
一节](http://www.boost.org/doc/libs/1_65_1/libs/type_traits/doc/html/index.html)

## 4. 只在参数类有指定成员函数名定义的情况下调用该成员函数

回到最开始的想法，假设我们希望只有传入的对象，有`logid`这个成员函数时就调用logid，应该怎么做？

```cpp
#include <iostream>
#include "boost/tti/has_member_function.hpp"

//BOOST_TTI_HAS_MEMBER_FUNCTION宏会生成has_member_function_logid函数
BOOST_TTI_HAS_MEMBER_FUNCTION(logid);

//foo zoo定义了void logid()函数
struct foo {
    void logid() { std::cout << "foo" << std::endl; }
};//foo

struct bar {
};//bar

struct zoo {
    void logid() { std::cout << "zoo" << std::endl; }
};//zoo

//第二个参数通过enable_if指定只有存在成员函数logid下才有效
template <class T, class = typename std::enable_if<has_member_function_logid<T, void>::value>::type>
void dohana(T t) {
    t.logid();
}

int main() {
    //测试has_member_function函数
    std::cout << has_member_function_logid<foo, void>::value << std::endl;//true
    std::cout << has_member_function_logid<bar, void>::value << std::endl;//false
    std::cout << has_member_function_logid<zoo, void>::value << std::endl;//true
    std::cout << has_member_function_logid<zoo, int>::value << std::endl;//false

    dohana(foo());//foo
    dohana(zoo());//zoo
    //compiler error:error: no type named ‘type’ in ‘struct std::enable_if<false, void>’
    // dohana(bar());
    return 0;
}
```

这样我们就实现了在编译期间调用指定函数，而不是运行期间通过反射来做。

## 5. 更多应用

实际上SFINAE在大型项目里非常常见，例如[protobuf](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/compiler/cpp/cpp_helpers.h)有大量使用:

```cpp
  ...
  template <typename I, typename = typename std::enable_if<
                            std::is_integral<I>::value>::type>
  static std::string ToString(I x) {
    return StrCat(x);
  }
```

## 6. 参考

1. [Substitution failure is not an error
](https://en.wikipedia.org/wiki/Substitution_failure_is_not_an_error)
2. [An introduction to C++'s SFINAE concept: compile-time introspection of a class member](https://jguegant.github.io/blogs/tech/sfinae-introduction.html)
3. [boost Metaprogramming](http://www.boost.org/doc/libs/1_65_1/libs/libraries.htm#Metaprogramming)
