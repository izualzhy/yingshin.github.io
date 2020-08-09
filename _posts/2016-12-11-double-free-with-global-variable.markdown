---
title: 追core笔记之四：由FileDescriptorTables::~FileDescriptorTables看静态库共享库的全局变量double-free的问题
date: 2016-12-11 11:21:09
excerpt: "由FileDescriptorTables::~FileDescriptorTables看静态库共享库的全局变量double-free的问题"
tags: core  global
---

最近把负责模块依赖的protobuf版本从2.4升级到了2.6，程序运行时正常，但是退出的时候，一个诡异的core出现了，部分core栈如下：

```cpp
#12 0x00000000006cba46 in ~hash_map (this=0xe98ab0 <google::protobuf::FileDescriptorTables::kEmpty+208>, __in_chrg=<optimized out>)
#13 0x00000000006cba46 in google::protobuf::FileDescriptorTables::~FileDescriptorTables (this=0xe989e0 <google::protobuf::FileDescriptorTables::kEmpty>, __in_chrg=<optimized out>)
#14 0x0000003f0b030f4b in __cxa_finalize () from /lib64/tls/libc.so.6
#15 0x00007f9673931a13 in __do_global_dtors_aux () from ./depends/lib/liblinkschema.so
```

查看pb2.6源码`kEmpty`的定义

```cpp
class FileDescriptorTables
  ...
  // Empty table, used with placeholder files.
  static const FileDescriptorTables kEmpty;
```

看上去是一个静态变量的重复析构的问题，而`kEmpty`变量正是pb2.6新引入的。本文接下来将分析下原因并找到对应的几种解决办法。

<!--more-->

### 1. 简化问题

模块依赖混用了静态库和动态库，我们首先试着完成一个复现core的简化demo版本。

结构如下：

```cpp
# 静态库:libfoo.a，定义了static变量(类static，等价于namespace下的全局static变量)
# 动态库:libfoo.so
    static link libfoo.a
# 可执行文件:main
    static link libfoo.a
    dynamic link libfoo.so
```

首先是静态库libfoo.a的接口文件foo.h：

```cpp
#ifndef _FOO_H
#define _FOO_H

#include <stdio.h>
#include <string>

class Foo {
public:
    static Foo s_foo;

    Foo();
    ~Foo();
    void foo();
private:
    std::string _data;
};
#endif  //_FOO_H
```

实现放在foo.cpp

```cpp
#include "foo.h"

Foo Foo::s_foo;

Foo::Foo() :
    _data("foo") {
        printf("Foo  this:%p\n", this);
    }

Foo::~Foo() {
    printf("~Foo this:%p\n", this);
}

void Foo::foo() {
    printf("foo  this:%p\n", this);
}
```

接着是动态库的接口和实现：

```cpp
//bar.h
#ifndef _BAR_H
#define _BAR_H

extern "C" {
    void bar();
}

#endif  //_BAR_H
```

```cpp
//bar.cpp
#include "foo.h"
#include "bar.h"

void bar() {
    Foo::s_foo.foo();
}
```

`main`的实现如下：

```cpp
#include <iostream>
#include "foo.h"
#include "bar.h"

int main() {
    std::cout << "enter main" << std::endl;
    Foo::s_foo.foo();
    bar();
    std::cout << "leave main" << std::endl;

    return 0;
}
```

贴下makefile：

```makefile
main:main.cpp libfoo.a libbar.so
        g++ -o main main.cpp -L. -lfoo -lbar -g

libfoo.a:foo.cpp
        g++ -c -fPIC $<
        ar crv $@ foo.o

libbar.so:bar.cpp libfoo.a
        g++ -c -fPIC bar.cpp
        g++ -shared -o $@ bar.o -L. -lfoo
```

执行main：

```
Foo  this:0x601808
Foo  this:0x601808
enter main
foo  this:0x601808
foo  this:0x601808
leave main
~Foo this:0x601808
~Foo this:0x601808
*** Error in `./main': double free or corruption (fasttop): 0x0000000000602040 ***
...
7f1bfa9c1000-7f1bfa9c2000 r-xp 00000000 07:05 3736227                    /home/users/y/Training/test/libtest/so_should_with_no_static_lib/libbar.so
7f1bfa9c2000-7f1bfabc2000 ---p 00001000 07:05 3736227                    /home/users/y/Training/test/libtest/so_should_with_no_static_lib/libbar.so
7f1bfabc2000-7f1bfabc3000 rw-p 00001000 07:05 3736227                    /home/users/y/Training/test/libtest/so_should_with_no_static_lib/libbar.so
...
```

可以推断`0x601808`是`Foo::s_foo`的地址，同时猜测在`main libbar.so`分别初始化和析构了两次（这是否是我们想要的？），而core的最直接原因在于是对同一块内存操作，因此出现了double free的问题。

实际上我在升级pb2.4到2.6遇到的这个double-free的问题，在pb的github上也有人提过[issue](https://github.com/google/protobuf/issues/194)。

接下来我们从demo分析下进一步的原因。

### 2. 为什么会double-free

首先看下main产生的部分core栈：

```
#7  0x0000000000400f87 in Foo::~Foo() ()
#8  0x00007f1bf9e3023f in __cxa_finalize () from /top/lib/libc.so.6
#9  0x00007f1bfa9c1b06 in __do_global_dtors_aux ()
   from /home/users/y/Training/test/libtest/so_should_with_no_static_lib/libbar.so
#10 0x00007f1bfade1000 in ?? ()
#11 0x0000000000000001 in ?? ()
#12 0x00007fffcd1a5450 in ?? ()
#13 0x00007f1bfa9c1d61 in _fini ()
   from /home/users/y/Training/test/libtest/so_should_with_no_static_lib/libbar.so
#14 0x00007fffcd1a5450 in ?? ()
#15 0x00007f1bfabd1e38 in _dl_fini () from /top/lib64/ld-linux-x86-64.so.2
```

我们知道`ld-xxx.so`就是linux下的动态链接器，同时对比`__do_global_dtors_aux _fini`这些函数的地址，和位于前面`libbar.so`的mmap内存区间可以推断我们的初步想法：

**`libbar.so`卸载时尝试释放`s_foo`内存时出core。**

从前面的stdout可以看到，`s_foo`的地址为`0x601808`。

这段地址位于main的.bss段:

```
$ nm main | c++filt  | grep s_foo
0000000000601808 B Foo::s_foo
```

`libbar.so`卸载的时候尝试释放这块内存，因此引起了double-free。

目前完整的推断如下：

`foo.o`里的`s_foo`被链接到了`main`和`libbar.so`，因此共有两个`s_foo`对象存在，所以会构造和析构两次，那么问题来了，这两个对象为什么会有相同个地址？

这个问题要从地址无关代码（Postion-independent Code）说起，我们知道**共享对象在编译时不能假设自己在进程虚拟地址空间中的位置**，而对编译器来讲，产生地址无关代码要采用相对偏移的寻址方式。对于模块间的数据访问，ELF的做法是在数据段里面建立一个指向这些变量的指针数组，也被称为全局偏移表(Global Offset Table, GOT)，当代码需要引用该数据时，可以通过GOT中相对应的项间接引用。

看下demo里的.GOT段

```
$ objdump -R libbar.so | c++filt  | grep s_foo
0000000000201228 R_X86_64_GLOB_DAT  Foo::s_foo
objdump -R main | c++filt  | grep s_foo
0000000000601620 R_X86_64_GLOB_DAT  Foo::s_foo
```

而对于`s_foo`这种全局变量，实现fPIC更复杂一些，因为`main`在它的.bss段创建了`s_foo`，而.GOT对应项的指针指向该地址，例如我们前面看到的地址`0000000000601808`，可以通过`objdump -d -S main`看下main对应的汇编验证下：

```
    Foo::s_foo.foo();
  400e40:|  bf 08 18 60 00       |  mov    $0x601808,%edi
  400e45:|  e8 60 01 00 00       |  callq  400faa <_ZN3Foo3fooEv>
```

可以看到`Foo::s_foo`的this指针被set成`0x601808`。

`libbar.so`静态链接了`libfoo.a`，因此负责了`s_foo`的生命周期，对于`main`也是一样，但是两者都指向了main的`bss`段，同时动态库与静态库不同的一点的是：动态库的加载和卸载会负责自身内存的分配和释放。

### 3. 解决方案

#### 3.1. 代码风格

在谈解决方案前，实际上更想先说下代码风格，全局变量/静态变量都应当是POD（plain old data）类型的。G厂的代码规范明确规定了[这点](https://google.github.io/styleguide/cppguide.html#Static_and_Global_Variables)

> Objects with static storage duration, including global variables, static variables, static class member variables, and function static variables, must be Plain Old Data (POD): only ints, chars, floats, or pointers, or arrays/structs of POD.

实际上我厂也有对应的代码要求哈哈哈哈。

但是protobuf的代码还有这个问题（看来代码风格这个在G厂一样是问题啊。。。），不过protobuf最新版已经修正了这个问题，通过一个`static const`的接口实现：

```
class FileDescriptorTables {
  // Empty table, used with placeholder files.
  // 去掉了2.6.1里的这句：static const FileDescriptorTables kEmpty;
  inline static const FileDescriptorTables& GetEmptyInstance();
}

inline const FileDescriptorTables& FileDescriptorTables::GetEmptyInstance() {
  InitFileDescriptorTablesOnce();
  return *file_descriptor_tables_;
}

inline void InitFileDescriptorTablesOnce() {
  ::google::protobuf::GoogleOnceInit(
      &file_descriptor_tables_once_init_, &InitFileDescriptorTables);
}
```

#### 3.2. 编译链接

根据上面分析的原因，我们可以考虑两种对应的解决方案。

1. 只保留一个`s_foo`  
2. 保证两个`s_foo`不冲突  

先看下方案一

这个方案的前提是我们需要的是同一个`s_foo`对象，不论在动态库、可执行文件里都只保留一个。

而链接的过程就是不断解决`undefined symbol`的一个过程：从静态库加载未定义的符号，动态库则全部加载。对`main`来讲，`s_foo`就是一个未定义的符号，从`libfoo.a`加载，可以改成从`libbar.so`加载。

因此我们可以这么修改`makefile`

```makefile
main:main.cpp libfoo.a libbar.so
        g++ -o main main.cpp -lbar -L. -lfoo -g
```

动态库调整到静态库前面，因此只有一个`s_foo`对象。

甚至demo里我们可以去掉`-lfoo`。

或者`libbar.so`里不要链接`libbar.a`，因为动态库是允许`undefined symbol`的。

```makefile
libbar.so:bar.cpp
        g++ -fPIC -shared -o libbar.so $<
```

类似的`libfoo.a`改成动态库也可以解决问题

接着看下方案二

makefile的修改方式

```makefile
libbar.so:bar.cpp
        g++ -fPIC -shared -Wl,-Bsymbolic -o libbar.so $< -L. -lfoo
```


`libbar.so`和`main`各自负责自己的`s_foo`对象

```
Foo  this:0x7fc1058aa2c0
Foo  this:0x601808
```

从maps的内存映射可以看到`0x7fc1058aa2c0`正是位于`libbar.so`的映射区间

```
7f1d7d7fb000-7f1d7d7fc000 r-xp 00000000 07:05 3736228                    /home/users/y/Training/test/libtest/so_should_with_no_static_lib/libbar.so
7f1d7d7fc000-7f1d7d9fc000 ---p 00001000 07:05 3736228                    /home/users/y/Training/test/libtest/so_should_with_no_static_lib/libbar.so
7f1d7d9fc000-7f1d7d9fd000 rw-p 00001000 07:05 3736228                    /home/users/y/Training/test/libtest/so_should_with_no_static_lib/libbar.so
```

可以看下加了`-Wl, -Bsymbolic`的`libbar.so`的.GOT段没有`Foo::s_foo`。

`-Bsymbolic`使得动态库的内部符号都本地化了。但是有一些工程经验表明该编译选项可能会有一些影响：<https://software.intel.com/en-us/articles/performance-tools-for-software-developers-bsymbolic-can-cause-dangerous-side-effects/>

因此更推荐采用修改`foo.h`的方式实现各自拥有`s_foo`的目的，可以不用修改makefile：

```
__attribute__ ((visibility ("hidden"))) static Foo s_foo;
```

### 4. GOT与PLT

再多扯两句关于GOT和PLT的部分

GOT（Golbal Offset Table）多用于实现地址无关代码。  
PLT（Procedure Linkage Table）目标是为了减少动态库与静态库间的性能差距，也就是延迟绑定(Lazy Binding)，函数第一次被用到时才进行绑定。

举个例子

```
#include <stdio.h>

int main() {
    puts("put called.");

    return 0;
}
```

`g++  -g -Wall -Werror -std=c++11 -lpthread  test_test.cpp   -o test_test`生成`test_test`执行文件，看下.GOT段

```
$ objdump -R test_test | c++filt 

test_test:     file format elf64-x86-64

DYNAMIC RELOCATION RECORDS
OFFSET           TYPE              VALUE 
0000000000600aa0 R_X86_64_GLOB_DAT  __gmon_start__
0000000000600ac0 R_X86_64_JUMP_SLOT  __gmon_start__
0000000000600ac8 R_X86_64_JUMP_SLOT  puts
0000000000600ad0 R_X86_64_JUMP_SLOT  __libc_start_main
```

看到`0x600ac8`是`puts`在.GOT段的地址，接着gdb看下`puts`前后该地址内容发生了什么变化

```
Breakpoint 1, main () at test_test.cpp:4
4           puts("put called.");
(gdb) x /xg 0x600ac8
0x600ac8 <puts@got.plt>:        0x0000000000400516
(gdb) n
put called.
6           return 0;
(gdb) x /xg 0x600ac8
0x600ac8 <puts@got.plt>:        0x00007ffff7060c80
(gdb) info symbol 0x00007ffff7060c80
puts in section .text of /top/lib/libc.so.6
```

可以看到`puts`第一次被调用后，`0x600ac8`存储的内容为`0x00007ffff7060c80`，即真正的`libc.so.6`的`puts`的地址。感兴趣的同学可以继续gdb看下，第一次调用会逐步调用到

```
_dl_runtime_resolve in section .text of /top/lib64/ld-linux-x86-64.so.2
```

也就是动态加载器的部分。
