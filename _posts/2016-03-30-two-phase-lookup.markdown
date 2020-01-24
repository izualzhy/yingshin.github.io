---
title:  "C++里的Two phase lookup"
date: 2016-03-30 22:19:03
excerpt: "Two phase lookup"
tags: [compliler]
---

本文主要介绍C++里的two phase lookup，网上关于two phase lookup的文章很多，这里主要介绍下自己整理之后的一些看法。

<!--more-->

首先从下面两个例子说起：

```
//1st
template <class T>
class A {
protected:
    int a;
};

template <class T>
class B : public A<T> {
public:
    void foo() {
        a = 0;//error: 'a' was not declared in this scope
    }
};

int main() {
    B<int> b;
    b.foo();

    return 0;
}
```

```
//2st
template <typename T>
class Foo {
public:
    void this_will_compile_right() {
        this->is->valid->code. get->off->my->lawn;
    }
    /*
    void this_will_compile_error() {
        mine->is->valid->code. get->off->my->lawn;
    }
    */
    void fun() {
        std::cout << "Foo::fun" << std::endl;
    }
};

int main() {
    Foo<int> foo;
    foo.fun();

    return 0;
}
```

上面两段代码的正确性稍后分析，先说下模板的编译过程。  

我们知道模板是在用到的时候才实例化的，比如这段代码：

```
template <class T>
class A {
public:
    void f() {}
    void g() {}
};

int main() {
    A<int> a;
    a.f();

    return 0;
}
```

编译后的符号只有这个：`W A<int>::f()`，注意：编译器很聪明的没有生成`A::g`。  

这个过程细分的话，就是Two phase name lookup了。  
1. 模板定义阶段：这个时候编译器会检查语法，例如是否少了`;`。查找模板中独立的名字(all non-dependent names are resolved (looked up))  
2. 模板实例化阶段：查找依赖的名字（dependent names are resolved.）  

独立的名字就是指该名字不依赖模板参数，反之就是依赖名字。  
例如第一个例子里`a`就是一个独立的名字，而如果使用了`T::Type b`，那么b就是一个依赖的名字，因为Type依赖T这个模板参数。
`this`也是一个依赖的名字。  
因此`a`会放到阶段1查找。  
而`b this`会放到阶段2查找。  

具体看下例子1，编译的错误贴在源码里了。  
编译器在阶段1想要查找a这个名字，但是当前还没有实例化A，因此也就没法找a这个名字。所以报错：  

```
error: 'a' was not declared in this scope
```

改成`this->a = 0`后就可以编译过了，因为把查找a这个名字的动作推迟到了阶段2，此时已经实例化，可以从基类里找到对应的名字，因此正确。

**那么为什么会有Two phase name lookup这个功能呢？**  

实际上有些编译器是一些不支持的，根据我的理解，如果所有检查都放到实例化的时候，可能导致编译的错误不够明确，因此对于一些语法(syntax)的检查放到了阶段1，而语义(semantic)的检查则放到了阶段2.  

阶段2也是必须的，语义的检查并不能统一放到阶段1.  比如例子1里，特化模板可能的确没有`a`这个名字。  
比如下面的例子可以正常编译：

```
template <class T>
class Base {
public:
};

template <class T>
class Derived : public Base<T> {
public:
    void foo() {
        this->a = 0;
    }
};

template <>
class Base<int> {
public:
    int a;
};

int main() {
    Derived<int> a;
    a.foo();

    return 0;
}
```

介绍完这些，再来看下例子2。  
我们知道如果Foo不是模板的话肯定是编译不过的。  
但是注释放开前这段代码是可以编译通过的，解释下原因：  
this是一个dependent name，因此这行放到了阶段2来查找名字，但在阶段2，这个函数`this_will_compile_right`并没有用到，因此实例化的类里自动跳过了这个函数，所以可以编译通过。 注释放开后阶段1查找名字失败，因此报错。  

与Two phase name lookup相关的还有编译时常见的`Dependent scope and nested templates`这个错误。

例如这段代码：

```
template <typename T>
struct Foo {
    Foo() {
        T::Frob (x);
    }
};

int main() {

    return 0;
}
```

尽管没有任何实例化的类，编译仍然报错：

```
test_template3.cpp: In constructor 'Foo<T>::Foo()':
test_template3.cpp:22:9: error: need 'typename' before 'T:: Frob' because 'T' is a dependent scope
         T::Frob x;
         ^
test_template3.cpp:22:17: error: expected ';' before 'x'
         T::Frob x;
                 ^
```

这是为了帮助编译器在阶段1的编译工作，就需要明确告诉编译器Frob是一种类型定义（如果的确是的话），具体可以参考这里：[Dependent scope and nested templates](http://stackoverflow.com/questions/6571381/dependent-scope-and-nested-templates/6571836#6571836)。  

### 参考
1. [古怪的 C++ 问题](http://blog.codingnow.com/2010/01/cpp_template.html)  
2. [Why can't I use variable of parent class that is template class?](http://stackoverflow.com/questions/10171242/why-cant-i-use-variable-of-parent-class-that-is-template-class)  
3. [Template instantiation details of GCC and MS compilers](http://stackoverflow.com/questions/7182359/template-instantiation-details-of-gcc-and-ms-compilers/7241548#7241548)  
4. [Two phase lookup - explanation needed](http://stackoverflow.com/questions/7767626/two-phase-lookup-explanation-needed?lq=1)
