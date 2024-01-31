---
title: 追core笔记之三：扯扯智能指针
date: 2016-11-26 23:30:39
excerpt: "追core笔记之三：扯扯智能指针"
tags: core
---

[上篇文章](http://izualzhy.cn/how-to-write-safe-callback)介绍了在回调时使用智能指针。可能因为智能指针太智能了，使用起来需要思考的很少。本文试图稍微总结一下“使用很自然但仔细想想很巧妙”的地方，所有例子均使用`boost::shared_ptr`。

<!--more-->

### 1. 类型转换

智能指针之间的类型转换遵循原始指针之间的转换原则，如果原始指针可以互相转换，智能指针就可以。

```cpp
class Base {
};//Base

class Derived : public Base {
};//Derived

int main() {
    {
        boost::shared_ptr<Base> pBase(new Base);
        // boost::shared_ptr<Derived> pDerived(pBase); Error
    }
    {
        boost::shared_ptr<Derived> pDerived(new Derived);
        boost::shared_ptr<Base> pBase(pDerived);
    }

    return 0;
}
```

`boost::shared_ptr<Derived> pDerived(pBase);`编译错误，因为基类指针无法强转为子类指针。

摘抄了构造函数相关的源码

```cpp
template<class T> class shared_ptr
{
    template<class Y>
    explicit shared_ptr( Y * p ): px( p ), pn() // Y must be complete
    {
        boost::detail::sp_pointer_construct( this, p, pn );
    }

    template<class Y>
    shared_ptr( shared_ptr<Y> const & r )
    BOOST_NOEXCEPT : px( r.px ), pn( r.pn )
    {
        boost::detail::sp_assert_convertible< Y, T >();//能够转换
    }

    typedef typename boost::detail::sp_element< T >::type element_type;
    element_type * px;                 // contained pointer
    ...
```

构造函数的Y类型需要能够强制转换为定义的T类型。

`sp_assert_convertible`实现在编译时断言

```cpp
template< class Y, class T > inline void sp_assert_convertible()
{
#if !defined( BOOST_SP_NO_SP_CONVERTIBLE )
    // static_assert( sp_convertible< Y, T >::value );
    typedef char tmp[ sp_convertible< Y, T >::value? 1: -1 ];
    (void)sizeof( tmp );
#else
    T* p = static_cast< Y* >( 0 );//编译时检查能否转换
    (void)p;

#endif
}
```

### 2. px/pn是什么

在gdb时，`print`一个智能指针我们得到这样的结果：

```cpp
$1 = {
  px = 0x603030,
  pn = {
    pi_ = 0x603010
  }
}
```

而我们的目的往往是想看下指针位置或者指向的对象的值。`px pn`分别代表什么呢？

```cpp
    element_type * px;                 // contained pointer
    boost::detail::shared_count pn;    // reference counter
```

`element_type * px`保留了传入的原始指针  
`boost::detail::shared_count pn`包装了`sp_counted_base * pi_`，指向同一块内存的所有`shared_ptr`共同维护`pi_`指向的内存，也就是负责记录`use_count weak_count`的部分，并在引用计数降到0时析构原始指针。

例如`shared_count`重载`operator =`的源码可以对应出上面的设计：

```cpp
    shared_count & operator= (shared_count const & r) // nothrow
    {
        sp_counted_base * tmp = r.pi_;

        if( tmp != pi_ )
        {
            if( tmp != 0 ) tmp->add_ref_copy();
            if( pi_ != 0 ) pi_->release();
            pi_ = tmp;
        }

        return *this;
    }
```


### 3.定制化的析构

上段代码中的`pi_->release()`可能会调用`dispose`

```cpp
    virtual void dispose() // nothrow
    {
        del( ptr );
    }
```

`del`在构造函数里传入

```cpp
    sp_counted_impl_pd( P p, D & d ): ptr( p ), del( d )    {    }
    sp_counted_impl_pd( P p ): ptr( p ), del()    {    }
```

`sp_counted_impl_pd`是`sp_counted_base`的子类，在`shared_counter`构造，再上层则是`shared_ptr`的构造函数

```cpp
    template<class Y, class D> shared_ptr( Y * p, D d ): px( p ), pn( p, d )
    {
        boost::detail::sp_deleter_construct( this, p );
    }
```

定制化的析构`d`在某些场景下非常有用，比如我们除了`delete`指针本身，还有一些其他清理的工作，例如容器的`erase`。同时使得`shared_ptr`不仅能够管理内存，而是提升为一个资源管理的神器。

例如我们可以这样打开文件而不用担心句柄的泄露。

```cpp
boost::shared_ptr<FILE> fp(fopen("./1.txt", "r"), fclose);
```

### 4. delete不见了，那new呢？

使用智能指针，我们几乎可以消除显式的`delete`调用，但是代码里还有很多`new`，看着总是有些不对称。

`smart_ptr`提供了一个工厂函数`make_shared`，可以接收若干参数new对象出来，借助于`auto`关键字，我们可以写出这样的代码：

```cpp
#include "boost/smart_ptr.hpp"

auto pfoo = boost::make_shared<Foo>("xiaoming", 6);
auto pv = boost::make_shared<std::vector<int> >(10, 2);
```

### 5. 线程安全

一个`shared_ptr`可以被多个线程安全读取，其他访问形式结果未定义。

### 6. 前置声明

智能指针的声明允许前置声明，例如

```cpp
class Foo;
typedef boost::shared_ptr<Foo> FooSharedPtr;
```

但是定义时，必须是`complete`的，这点我觉得很正常，因为一般定义`shared_ptr`的时候需要类型的构造函数。

而在`shared_ptr`里需要类型定义更根本的原因是：如果`pi_`new失败，那么需要delete原指针。

实现编译时断言的代码如下

```cpp
template<class T> inline void checked_delete(T * x)
{
    // intentionally complex - simplification causes regressions
    typedef char type_must_be_complete[ sizeof(T)? 1: -1 ];
    (void) sizeof(type_must_be_complete);
    delete x;
}
```

### 7. 智能指针的引用

在不需要传值的情况下，尽量用`const boost::shared_ptr<Class>& class_pointer;`

### 8. unique

有时候你需要确认下当前是否只有自己持有了原始指针的内存管理权，可以使用`unique`接口

```cpp
//shared_ptr
    bool unique() const BOOST_NOEXCEPT {
        return pn.unique();
    }

//shared_count
    bool unique() const {
        return use_count() == 1;
    }
```

`use_count`是当前的引用计数，有的书里介绍该接口效率不高，且有时候不可用。看了下源码，没有找到原因。

### 9. 交叉引用

不要试图写出这样的代码：

```cpp
class B;

class A {
public:
    A() { std::cout << "A::A" << std::endl; }
    ~A() { std::cout << "A::~A" << std::endl; }
    boost::shared_ptr<B> _b;
};

class B {
public:
    B() { std::cout << "B::B" << std::endl; }
    ~B() { std::cout << "B::~B" << std::endl; }
    boost::shared_ptr<A> _a;
};
```

会导致内存泄露，例如

```cpp
        boost::shared_ptr<A> a(boost::make_shared<A>());
        boost::shared_ptr<B> b(boost::make_shared<B>());
        a->_b = b;
        b->_a = a;
```

退出作用域时，new的两块内存都不会正确的析构。

退出时指向`new A/new B`内存的智能指针引用计数都等于2，当`b`析构时，指向`new B`引用计数减1，当`a`析构时，指向`new A`的引用计数减1，但此时两块内存的引用计数都还是1。

我在实践中没有遇到过这种代码，因此记录下网上较多的建议修改方案：修改某个类的成员变量为`weak_ptr`。

### 10. shared_ptr<void>

`shared_ptr<void>`能够存储`void*`的指针，而`void*`可以指向任意类型，因此我们可以通过`shared_ptr<void>`指向任意一块内存，原因可以参考[How does shared_ptr<void> know which destructor to use?](https://stackoverflow.com/questions/57344571/how-does-shared-ptrvoid-know-which-destructor-to-use)。而`unique_ptr<void>`会[导致编译错误](https://stackoverflow.com/questions/39288891/why-is-shared-ptrvoid-legal-while-unique-ptrvoid-is-ill-formed?rq=1)。

不关心应用数据类型的场景比较有用，例如异步线程里的上下文默认为`shared_ptr<void>`，具体的转化为各种不同类型，则在应用层决定。

### 11. 同一内存的不同offset

对同一块内存，不同的`shared-ptr`可以管理不同 offset，例如一段字符串，传入不同接口时，提供不同的 substr.

```cpp
    std::shared_ptr<char> p(new char[1024]);
    strncpy(p.get(), "hello world.", 1024);
    //p:hello world. use_count:1
    printf("p:%s use_count:%ld\n", p.get(), p.use_count());

    std::shared_ptr<char> p2(p, p.get() + 5);
    //p:hello world. use_count:2
    printf("p:%s use_count:%ld\n", p.get(), p.use_count());
    //p2: world. use_count:2
    printf("p2:%s use_count:%ld\n", p2.get(), p2.use_count());
```

### 12. 其他

`boost::shared_ptr`还提供了其他一系列方便的接口。

比较运算，测试两个`shared_ptr`是否相等。同时提供`operator<`，因此可以用于标准关联容器。

以及`operator<<`用于输出内部的指针值，`owner_before`等接口。

另外还有别名构造函数，`owner_less`等，包括各种类型转换的接口，例如`dynamic_pointer_cast/static_pointer_cast`。

这些我用的不多，在此只是记录一下。

