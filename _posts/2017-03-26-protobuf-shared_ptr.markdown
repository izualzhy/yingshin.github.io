---
title: 再扯扯智能指针之protobuf
date: 2017-3-26 17:33:13
excerpt: "再扯扯智能指针之protobuf"
tags: protobuf
---

一直对智能指针很感兴趣，之前写过一篇[扯扯智能指针](http://izualzhy.cn/smart_pointer)，平时没事也会翻出来boost chrome源码里的智能指针学习下。

在这个过程中发现一个有趣的现象，很多项目在选择开源的时候，为了减少用户使用的成本，往往希望尽量少的@第三方代码，特别是有些源码比较庞大、编译耗时的第三方库，例如boost。我厂开源的sofa-rpc里为了不依赖boost同时又使用智能指针，就有大量的[smartptr代码](https://github.com/baidu/sofa-pbrpc/tree/master/src/sofa/pbrpc/smart_ptr)，看了下感觉是从boost拿过来的，不过本身已经依赖了boost，有些画蛇添足。

个人比较推荐的是protobuf里的做法，因为c11已经默认支持了`std::shared_ptr std::enable_shared_from_this`等，如果用户的编译环境是c11，那么直接使用std原生的智能指针，否则使用模块里或者第三方的代码，比如boost。

protobuf里的智能指针是自己实现的，比较简单（只有500行），相比boost更具有可读性，当然缺点是提供的并不全，比如`make_shared`。

本文主要介绍下protobuf里智能指针的源码实现，包括`shared_ptr weak_ptr enable_from_this`，protobuf版本为2.6.0，代码位于`google/protobuf/stubs/shared_ptr.h`。

<!--more-->

## 1. 使用std原生智能指针

如果是c11或者MSVC编译器，使用std原生指针，这点通过alias完成。

```
namespace google {
namespace protobuf {
namespace internal {
#if !defined(UTIL_GTL_USE_STD_SHARED_PTR) && \
    (defined(COMPILER_MSVC) || defined(LANG_CXX11))
#define UTIL_GTL_USE_STD_SHARED_PTR 1
#endif

#if defined(UTIL_GTL_USE_STD_SHARED_PTR) && UTIL_GTL_USE_STD_SHARED_PTR
using std::enable_shared_from_this;
using std::shared_ptr;
using std::static_pointer_cast;
using std::weak_ptr;
#else
//自定义的智能指针实现
#endif
}
}
}
```

如果没有定义，那么就使用protobuf的实现。

## 2. 智能指针的实现

那么接下来说下protobuf里的实现过程。

逐个介绍下
1. `SharedPtrControlBlock`：引用计数管理  
2. `shared_ptr`  
3. `weak_ptr`  
4. `enable_shared_from_this`  
5. `static_pointer_cast`：智能指针类型转换

### 2.1. google::protobuf::internal::SharedPtrControlBlock用于计数

智能指针自然少不了引用计数，计数通过`SharedPtrControlBlock`完成

```
class SharedPtrControlBlock {
  template <typename T> friend class shared_ptr;
  template <typename T> friend class weak_ptr;
 private:
  SharedPtrControlBlock() : refcount_(1), weak_count_(1) { }
  Atomic32 refcount_;
  Atomic32 weak_count_;
};
```

原始对象由智能指针接管后，会开辟一段内存用于计数。同一原始对象上的这些智能指针，无论是`shared_ptr`还是`weak_ptr`，都可以访问到这段内存，对引用计数进行加减操作。`refcount_ weak_count_`分别是当前`shared_ptr` `weak_ptr`的个数（`weak_count_`严格来讲是`weak_ptr`个数+1）。

注意这两个计数初始化值均为1，也就是只有真的要管理内存了，`SharedPtrControlBlock`才会创建出来。

### 2.2. google::protobuf::internal::shared_ptr

`shared_ptr`负责管理对象所在的内存，通过对`Copy Constructor/Operator =`等的封装，确保对象上增加`shared_ptr`计数加1，析构`shared_ptr`时计数减1，并在合适的时机释放原始指针和计数。因此包含两个成员变量：原始指针和`SharedPtrControlBlock`。

```
  T* ptr_;
  SharedPtrControlBlock* control_block_;
```

#### 2.2.1. 构造、复制构造以及operator=

空的`shared_ptr`不管理任何指针，同时`control_block_`也不会被new出来

```
shared_ptr() : ptr_(NULL), control_block_(NULL) {}
```

当传入一个raw pointer时，`control_block_`被new出来并且使用默认值，这也是`SharedPtrControlBlock`初始化时`refcount_`设置为1的原因：

```
  explicit shared_ptr(T* ptr)
      : ptr_(ptr),
        control_block_(ptr != NULL ? new SharedPtrControlBlock : NULL) {
    // If p is non-null and T inherits from enable_shared_from_this, we
    // set up the data that shared_from_this needs.
    MaybeSetupWeakThis(ptr);
  }
```

`MaybeSetupWeakThis`主要的作用是：

如果对象父类是`enable_from_this`类型，那么初始化`enable_from_this`的成员变量。否则什么都不干。这里的语法也比较有意思，实现了两个版本的`MaybeSetupWeakThis`


```
  void MaybeSetupWeakThis(enable_shared_from_this<T>* ptr);
  void MaybeSetupWeakThis(...) { }
```

其中"大众化"的版本什么都不做，`enable_shared_from_this`参数的版本我们放到本文后面讲。

复制构造函数有两种，主要是参数类型上的区别，既支持定义时的模板类型T，也支持其他类型U，对U的要求就是：**U可以隐式转化为T**。两者最终都调用`InitializeWithStaticCast`

```
  template <typename U, typename V>
  void InitializeWithStaticCast(const shared_ptr<V>& r) {
    if (r.control_block_ != NULL) {
      RefCountInc(&r.control_block_->refcount_);

      ptr_ = static_cast<U*>(r.ptr_);
      control_block_ = r.control_block_;
    }
  }
```

可以看到这里就是一个拷贝`ptr_ control_block_`以及加计数的过程。

`operator=`稍微复杂一点，因为还要考虑到之前的`ptr_`是否需要释放，主要通过一个临时变量 + `std::swap`完成，本质上则是把资源释放的判断放到了临时变量的析构里。

注意这里的计数是针对`refcount_`，增加`shared_ptr`并不会引起`weak_count_`变化。

### 2.2.2. 析构

`shared_ptr`在析构时需要考虑是否释放`ptr_`以及`control_block_`，直接看下代码：


```
  ~shared_ptr() {
    if (ptr_ != NULL) {
      //首先判断refcount_减1的值
      //如果为0，表示自己是最后一个own原始指针的智能指针，需要释放ptr_
      if (!RefCountDec(&control_block_->refcount_)) {
        delete ptr_;

        // weak_count_ is defined as the number of weak_ptrs that observe
        // ptr_, plus 1 if refcount_ is nonzero.
        if (!RefCountDec(&control_block_->weak_count_)) {
          delete control_block_;
        }
      }
    }
  }
```

`delete ptr_`的判断在代码里说明了，`weak_count_`重点说下：

`weak_count_`的值初始化为1，随着`weak_ptr`的构造和析构分别增减1。因为可以认为`ptr_`与`shared_ptr`个数相关，而`control_block_`则跟`shared_ptr/weak_ptr`个数都相关，严格来讲无论多少个`shared_ptr`，对`weak_count_`贡献都是+1，而`weak_ptr_`则每个贡献+1。上面的析构函数里当原始指针上的最后一个`shared_ptr`析构时，会对`weak_ptr_`贡献-1，此时是否`delete control_block_`则由`weak_ptr_`的个数决定。

boost里的注释就比较简单直接：

```
class sp_counted_base
{
private:
    ...

    int use_count_;        // #shared
    int weak_count_;       // #weak + (#shared != 0)
```

这样的设计是非常合理的，保证了`shared_ptr`与`weak_ptr_`共同管理`control_block_`，同时相比其他设计保证了自增自减的操作最少。

此外`shared_ptr`提供了`use_count reset unique`等常见接口。

### 2.3. google::protobuf::internal::weak_ptr

`weak_ptr`相当于指针的弱owner，包含两个成员变量

```
  element_type* ptr_;
  SharedPtrControlBlock* control_block_;
```

实现上跟`shared_ptr`比较类似，不同的是复制时增加的计数是`weak_count_`

```
  void CopyFrom(T* ptr, SharedPtrControlBlock* control_block) {
    //复制成员变量：ptr_ control_block_
    ptr_ = ptr;
    control_block_ = control_block;
    if (control_block_ != NULL)
      //增加引用计数
      RefCountInc(&control_block_->weak_count_);
  }
```

众所周知`weak_ptr`最常用的接口是`lock`，我们重点分析下实现

```
  shared_ptr<T> lock() const {
    shared_ptr<T> result;
    if (control_block_ != NULL) {
      Atomic32 old_refcount;
      do {
        old_refcount = control_block_->refcount_;
        //如果refcount_ == 0，表示没有shared_ptr在管理ptr_，ptr_已经释放
        if (old_refcount == 0)
          break;
        //while这句非常关键，注意这种race condition
        //最后一个shared_ptr将refcount_置0，然后析构ptr_
        //此时执行while前control_block_->refcount_ = 0， old_refcount = 1
        //因此除了Swap，Compare是必须的
      } while (old_refcount !=
               NoBarrier_CompareAndSwap(
                   &control_block_->refcount_, old_refcount,
                   old_refcount + 1));
      if (old_refcount > 0) {
        result.ptr_ = ptr_;
        result.control_block_ = control_block_;
      }
    }

    return result;
  }
```

解释下while判断条件的必要性：

`CompareAndSwap`的伪代码如下：

```
Atomic32 NoBarrier_CompareAndSwap(volatile Atomic32* ptr,
                                  Atomic32 old_value,
                                  Atomic32 new_value)
// Atomically execute:
      result = *ptr;
      if (*ptr == old_value)
        *ptr = new_value;
      return result;
```

**如果ptr指向的内存当前的值为old_value，那么替换为new_value，无论什么情况下都返回old_value。**

为什么必须要引入这个函数呢？

因为即使`control_block_->refcount_`是原子的，也不能保证在修改过程中没有其他线程在修改，因此

比如假设我们这么写

```
if (old_refcount > 0) {
  RefCountInc(&control_block_->refcount_);
}
```

如果在if判断条件满足之后，`RefCountInc`调用之前，持有该内存的`shared_ptr`析构(if判断满足条件)。

```
      if (!RefCountDec(&control_block_->refcount_)) {
        delete ptr_;
```

此时race condition就会出现，ptr_已经delete，但是`lock`还是返回了一个`shared_ptr`。而`CompareAndSwap`则保证了这点。参考下`lock`函数的注释：

```
  // Return a shared_ptr that owns the object we are observing. If we
  // have expired, the shared_ptr will be empty. We have to be careful
  // about concurrency, though, since some other thread might be
  // destroying the last owning shared_ptr while we're in this
  // function.  We want to increment the refcount only if it's nonzero
  // and get the new value, and we want that whole operation to be
  // atomic.
```

trick一点，比如我们这么写：

```
if (old_refcount > 0) {
  RefCountInc(&control_block_->refcount_);
}
if (control_block->refcount_ == 1) {
  //自己是最后一个，无效内存
} else {
  //有效？
}
```

这种方式不仅不够“优雅”，而且无法解决两个`weak_ptr`同时`lock`的情况。

### 2.4. google::protobuf::internal::enable_shared_from_this

`enable_shared_from_this`使用时采用继承的方式。

比如：

```
class LinkBaseReader : public boost::enable_shared_from_this<LinkBaseReader>
```

只包含一个成员变量：

```
  weak_ptr<T> weak_this_;
```

在`shared_ptr`一节里介绍过`MaybeSetupWeakThis`这个函数，实现上就是填充了`weak_this_`这个成员变量：

```
template<typename T>
void shared_ptr<T>::MaybeSetupWeakThis(enable_shared_from_this<T>* ptr) {
  if (ptr) {
    CHECK(ptr->weak_this_.expired()) << "Object already owned by a shared_ptr";
    ptr->weak_this_ = *this;
  }
}
```

`shared_from_this`则提供了接口返回`shared_ptr<T>`，也就是自身的智能指针。

```
  shared_ptr<T> shared_from_this() {
    CHECK(!weak_this_.expired()) << "No shared_ptr owns this object";
    return weak_this_.lock();
  }
```

这里重点说下为什么要使用`weak_ptr_`？如果要通过`shared_from_this`返回自身的智能指针，那么保存的成员变量无非有三种选择：raw-pointer/shared-pointer/weak-pointer。

1. 存储原始指针`T* ptr_`不可行：`shared_from_this`调用时返回`shared_ptr<T>(ptr_)`不可行，因为每个返回的`shared_ptr<T>`都会尝试释放`ptr_`  
2. 存储`shared_ptr_`不可行，因为一个对象不应该包含指向自己的智能指针作为成员变量，否则引用计数永远>1，这点和之前介绍的[交叉引用](http://izualzhy.cn/smart_pointer)很像。  
3. 存储`weak_ptr_`可行，不会固定增加一个引用计数，注意`weak_this_`是在`shared_ptr::MaybeSetupWeakThis`调用时初始化的，也就是`shared_ptr::shared_ptr(T* ptr)`，这也是为什么继承`enable_shared_from_this`的子类对象一定需要是heap上对象同时由`shared_ptr`管理生命周期。代码`weak_this_->expired`接口的作用正是为了判断是否符合上述条件。  

因此注意不要在构造函数/析构函数里调用`shared_from_this`，`weak_this_`还没有指向任何shared_ptr.

### 2.5. google::protobuf::internal::static_pointer_cast

`static_pointer_cast`用于智能指针之间的类型转换，单独说下这个的原因是有个比较有意思的语法

```
template <typename T, typename U>
shared_ptr<T> static_pointer_cast(const shared_ptr<U>& rhs) {
  shared_ptr<T> lhs;
  lhs.template InitializeWithStaticCast<T>(rhs);
  return lhs;
}
```

`InitializeWithStaticCast`是`shared_ptr`的成员函数

```
  template <typename U, typename V>
  void InitializeWithStaticCast(const shared_ptr<V>& r)
```

注意`lhs.template ...`这句，如果我第一次写上面那段代码，估计会写成

```
  lhs.InitializeWithStaticCast<T>(rhs);
```

不过这样编译器会报错，跟[two-phase-lookup](http://izualzhy.cn/two-phase-lookup)有关，具体可以参考[这里](http://www.aerialmantis.co.uk/blog/2017/03/17/template-keywords/)，简言之，就是编译器需要明确告诉它这是一个类型。
