---
title: "3 Goals for Better Code"
date: 2019-01-05 18:20:52
tags: Cpp-Seasoning  STL
---

这篇笔记是关于 Sean Parent 在2013年的一篇演讲，题目就叫做《3 Goals for Better Code》，听完之后有比较多的共鸣和触动，因此专门记录下来。

关于 Sean Parent其人：

>Sean Parent is a principal scientist and software architect for Adobe Photoshop and mobile imaging applications.

## 1. No Raw Loops

>A raw loop is any loop inside a function where the function serves purpose larger than the algorithm implemented by the loop

简言之，就是如果在一个函数内写了一个 for/while 循环，而函数本身功能比这个循环要多，那就要想办法替代这个循环。用现有的标准库，实现一个 general 的函数，甚至发明一个新算法都可以。

Raw Loop的缺点：

+ Difficult to reason about and difficult to prove post conditions  
+ Error prone and likely to fail under non-obvious conditions  
+ Introduce non-obvious performance problems  
+ Complicates reasoning about the surrounding code  

作者举了他在 google 工作时代码 review Chromium 源码的一个例子，一个函数包含了很多个 for/while loop，导致可读性很差(有同感).

```c++
void PanelBar::RepositionExpandedPanels(Panel* fixed_panel) {
  CHECK(fixed_panel);
  // First, find the index of the fixed panel.
  int fixed_index = GetPanelIndex(expanded_panels_, *fixed_panel);
  CHECK_LT(fixed_index, static_cast<int>(expanded_panels_.size()));
  // Next, check if the panel has moved to the other side of another panel.
  const int center_x = fixed_panel->cur_panel_center();
  for (size_t i = 0; i < expanded_panels_.size(); ++i) {
    Panel* panel = expanded_panels_[i].get();
    if (center_x <= panel->cur_panel_center() ||
        i == expanded_panels_.size() - 1) {
      if (panel != fixed_panel) {
        // If it has, then we reorder the panels.
        std::tr1::shared_ptr<Panel> ref = expanded_panels_[fixed_index];
        expanded_panels_.erase(expanded_panels_.begin() + fixed_index);
        if (i < expanded_panels_.size()) {
          expanded_panels_.insert(expanded_panels_.begin() + i, ref);
        } else {
          expanded_panels_.push_back(ref);
        }
      }
      break;
    }
  }
  ...
```

如何通过一系列的办法简化了这个100多行的代码，思路就是**No Raw Loops**，工具使用 STL。

然而，重点来了，这段代码还一直在[代码库](https://chromium.googlesource.com/chromiumos/platform/window_manager/+/636fb791f3c8e4079c99c36178d5e41d250655b2/panel_bar.cc)里，并且被 CI 了上去！隔着屏幕我都能感觉到视频里 Sean Parent 深深的怨念。他大概断断续续花了3天时间，才跟代码开发者解释清楚为什么修改后能够生效。但是被另外一个 reviewer 给拦住了💔

>you can't replace that loop with find_if followed by rotate.it's too tricky nobody knows what rotate does.

我这是第一次听到 google 代码审查的负面说法，不过没有更多的八卦了👐。

现在我们看下具体都用到了什么

### 1.1. rotate

抽象的一个操作是 rotate，例如把 0 1 这两个元素与2 3 4 5 6 7 8 9互换。如图所示:

![rotate_v1](assets/images/cpp-seasoning/rotate_v1.png)

如果尝试实现一个循环来完成的话，那就需要了解下`std::rotate`了

```
    std::vector<int> v{0, 1, 2, 3, 4, 5, 6, 7, 8, 9};
    auto iter = std::rotate(v.begin(), v.begin() + 2, v.end());
```

`std::rotate`支持指定区间`first, ..., n_first, ..., last`，rotate 后的区间为`n_first, ..., last, first, ..., n_first - 1`，返回`first`的迭代器位置。

例如把 2 3 4 5 6 重新排列为 5 6 2 3 4

`auto iter = std::rotate(v.begin() + 2, v.begin() + 5, v.begin() + 7);`

![rotate_v2](assets/images/cpp-seasoning/rotate_v2.png)

### 1.2. stable_partition

第二个例子抽象下如图

![stable](assets/images/cpp-seasoning/stable_partition.png)

即把符合某一条件的元素(图里是偶数)移动到某个位置，使用`std::stable_partition`可以完成

```
bool IsOdd(int i) {
    return (i%2) == 1;
}

int main() {
    std::vector<int> v{0, 1, 2, 3, 4, 5, 6, 7, 8, 9};

    auto iter = std::stable_partition(v.begin(), v.begin() + 5, IsOdd);
    std::cout << "after 1st stable_partition, *iter:" << *iter << std::endl;
    std::copy(
            v.begin(),
            v.end(),
            std::ostream_iterator<int>(std::cout, "\n"));

    iter = std::stable_partition(v.begin() + 5, v.end(), std::not1(std::function< bool(int)>(IsOdd)));
    std::cout << "after 2nd stable_partition, *iter:" << *iter << std::endl;
    std::copy(
            v.begin(),
            v.end(),
            std::ostream_iterator<int>(std::cout, "\n"));

    return 0;
}
```

这些函数的功能看似简单，然后就是借助这几个函数(`find_if` `lower_bound` `range`等)，代码变得非常简洁，对于熟悉这些函数作用的 rd 来讲，上手成本要低很多，代码读起来也有幸福感。

此外还有很多其他建议，例如


>1. Use const auto& for for-each and auto& for transforms
>2. Use lambdas for predicates, comparisons, and projections, but keep them short
>3. ...

If you want to improve the code quality in your organization, replace all of your coding guidelines with one goal:

**No Raw Loops**

## 2. No Raw Synchronization primitives

不要在直接使用诸如 `Mutex`、`Atomic`、`Semaphore`、`Memory Fence` 的同步原语.

直接操作的缺点就是你很有可能导致各种 race condition，例如这样的代码:

```c++
template <typename T>
class bad_cow {
    struct object_t {
        explicit object_t(const T& x) : data_m(x) { ++count_m; }
        atomic<int> count_m;
        T           data_m; };
    object_t* object_m;
 public:
    explicit bad_cow(const T& x) : object_m(new object_t(x)) { }
    ~bad_cow() { if (0 == --object_m->count_m) delete object_m; }
    bad_cow(const bad_cow& x) : object_m(x.object_m) { ++object_m->count_m; }

    bad_cow& operator=(const T& x) {
        if (object_m->count_m == 1) object_m->data_m = x;
        else {
            object_t* tmp = new object_t(x);
            --object_m->count_m;
            object_m = tmp;
        }
        return *this;
    }
};
```

我们应当尝试使用更多已有的封装来完成并发，例如`std::async`

```
std::string foo() {
    std::this_thread::sleep_for(std::chrono::seconds(2));
    return "foo";
}

int main() {
    auto start = std::chrono::high_resolution_clock::now();
    std::future<std::string> result(std::async(foo));

    std::cout << result.get() << std::endl;
    auto end = std::chrono::high_resolution_clock::now();
    auto int_ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    std::cout << int_ms.count() << "ms" << std::endl;;

    return 0;
}
```

多尝试已有的封装，例如 PPL(Parallel Patterns Library)、TBB、std 里的 packaged_task等。

## 3. No Raw Pointers

这一节的观点比较激进，先说下什么是 Raw Pointer

```
A pointer to an object with implied ownership and reference semantics
T* p = new T
unique_ptr<T>
shared_ptr<T>
```

作者认为，A shared pointer is as good as a global variable，但同样属于 Raw Pointer，同样有着全局变量有的问题。

智能指针只是解决了 mem leak 的问题，但是存储的对象随时可能被其他线程改变，这个是很危险的事情。(同意，所以实践中我使用线程池的习惯:同一个对象只有一个线程处理，使用智能指针的好处不用 care 不同分支 return 时的资源释放问题)。

分析过程参考 keynote 原文，最后作者给了一个类似这样的实现，保存了 const 的对象，使用智能指针也只是为了资源释放更方便。

```
class object_t {
  public:
    template <typename T>
    object_t(T x) : self_(make_shared<model<T>>(move(x))) { }
        
    friend void draw(const object_t& x, ostream& out, size_t position)
    { x.self_->draw_(out, position); }
    
  private:
    struct concept_t {
        virtual ~concept_t() = default;
        virtual void draw_(ostream&, size_t) const = 0;
    };
    template <typename T>
    struct model : concept_t {
        model(T x) : data_(move(x)) { }
        void draw_(ostream& out, size_t position) const
        { draw(data_, out, position); }
        
        T data_;
    };
    
   shared_ptr<const concept_t> self_;
};

using document_t = vector<object_t>;

void draw(const document_t& x, ostream& out, size_t position)
{
    out << string(position, ' ') << "<document>" << endl;
    for (auto& e : x) draw(e, out, position + 2);
    out << string(position, ' ') << "</document>" << endl;
}
```

封装用到的模板技巧，类似于之前写过的[boost::any](https://izualzhy.cn/boost-any-sample-implemention).

完整的[演讲视频及文档地址](https://sean-parent.stlab.cc/papers-and-presentations/#c-seasoning)

## 4. 后记

这个视频我反反复复看了好几遍，无论视频最后还是网站的评论里，很多人都希望作者能够出一本书，Better Code、Clean Code 这种。特别是第一节，对我而言，引起很多反思。在接手代码时，多个循环变量替换复制无疑是很让人头疼的，每次都要在纸上模拟运行多次才能彻底了解，如果用已知的更为熟悉的函数替换掉，无疑能节省其他人阅读的工作量。但是我们往往忽略了这一点，写各种框架、系统是造轮子，写很多无用的循环也是造轮子，只是大小的区别。
