---
title: saverng追core笔记
date: 2016-12-24 12:17:19
excerpt: "追core笔记"
tags: core
---

9月份以来，接手了一个模块，最大的问题是线上稳定性很差。每天出成百的core，追查起来非常辛苦，也是工作以来接手的最头疼的一个模块。因此尝试记录整个的心得，除了如何定位core，也想就这些坑谈下如何写一个更健壮的程序。程序代码比较久远，拿现在的标准和经验去衡量之前的代码明显是不公平的，因此本文并无比较之意，请勿对号入座。

先稍微说下接手时该模块的现状：

1. 每天产生上百个core，core没有read权限(你敢信？)  
2. core没有历史记录，因此无法通过版本代码diff来比较，面对的就是这个接近10万行代码的模块，文档总共两页  
3. 模块使用了3种方式输出日志，打印非常随意，经常打满硬盘  
4. 线上部署1000个实例，通过一个并不稳定的集群管理程序管理，表现在：没有内存、cpu占用曲线/上线时core文件会被管理程序清理掉/pstack时进程会被管理程序重启等  
5. 模块使用了很多本地编译的静态/动态库，版本没有打平  
6. 除了core，还存在程序hang住不处理数据的问题  
7. 有些测试用例本身是错误的  

<!--more-->

首先的思路是通过[core的分类来抓住问题的重点](http://izualzhy.cn/pstack)，然后发现异步回调时使用了一些wild pointer导致的，所以有了这篇总结：[如何写出安全的回调](http://izualzhy.cn/how-to-write-safe-callback)。模块大部分都使用了智能指针，防止了内存泄露的问题，追core的过程中总结了下[自己对于智能指针的理解](http://izualzhy.cn/smart_pointer)。升级protobuf到2.6的过程中，引入了一个新的core，根本原因还是动态库的滥用但是没有正常的发布，因此记录在了[由FileDescriptorTables::~FileDescriptorTables看静态库共享库的全局变量double-free的问题](http://izualzhy.cn/double-free-with-global-variable-in-static-library)，最后总结了下[core栈为什么会乱的原因](http://izualzhy.cn/why-the-code-stack-is-overflow)。这些问题都在追core的过程中一一遇到，个人稍微总结了下，隐藏了很多细节，希望对读者有用。

追core，很多时候是排查很多种可能的原因，最终定位的一个过程，时间会比较漫长，而且无聊。如果有同学遇到跟我类似的状态，建议一定要多总结，因为这个过程不像在写新的代码，如果不总结一下，可能自己的提升就更有限了。

心态要一直摆正，积极解决问题，对core零容忍未必是共识，但是自己要明确own的模块不应该有core。

其实模块本身代码放到今天来看，也有很多可取之处，比如effective c++里提到的vector使用swap来释放内存，大量的使用了智能指针防止内存泄露，回调使用了自定义的类似于tr1里的function/bind的形式，在阅读以及维护上都避免了潜在的问题。然而也有一些类似于这样的定义：

```
saver_global_t g_global_conf_data;
```

虽然`g_`开头，但是实际上是一个成员变量，或者在一个成员函数里使用了`del_keep_dlc l2_clear_dlc protect_vip_dlc`这几个变量，而实际上前两个是成员变量，最后一个莫名其妙定义成了全局变量，在一个几十个成员变量的类里面，不深入追查完全没有意识到代码会这么写，给读懂代码带来了很大障碍，或者是看到这样的声明

```
static MergePolicy* get_instance();
```

就会经常忽略对应的实现，而实现实际上可能并不是单例

```
MergePolicy* MergePolicy::get_instance() {
    static pthread_once_t once_policy;
    pthread_once(&once_policy, &MergePolicy::make_policy_ary);
    int index = random() % MAX_POLICY_INSTANCE;
    return g_policy_ary[index];
}
```

类似的问题还有很多，这里没有指责之意，毕竟是几年前的代码。从代码规范等角度，今天我们应当尽量避免这种歧义的代码。

以上为总结。
