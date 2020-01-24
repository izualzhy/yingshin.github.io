---
title: 避免使用虚函数作为动态库的接口
date: 2016-12-3 11:47:09
excerpt: "避免使用虚函数作为动态库的接口"
tags: vtable  lib
---

最近在微信上看到[一篇文章](http://mp.weixin.qq.com/s?timestamp=1480736663&src=3&ver=1&signature=dSLt*qzwsSv*QuBHHvhmIAXB9MD4jXdbDYYx6hAt5q11kEmQG4PVLmhopXImsrjPWs4uH-qlRfxOrPzUYg-1gP9kELDpZ3mNiW-ofRvLnGeTU9x76KeAaOTXegQpw9Ymo32oDJmSM0BfHQadmOSRYRo5pRKmgYVeBimNP1-gZp0=)，介绍了Google关于使用静态库or动态库的习惯。

> 另外，出于Google的习惯（C++亦是如此），Go语言默认采用static linking的方式生成binary。这意味着部署的时候无需担心服务器上是否有外部依赖的动态链接库，只需一个binary一切搞定。

一方面我们享受着动态库升级带来的便利，比如Qt，随处可见的各种dll，另一方面又因为so的改动而带来二进制不兼容的问题。一直想把工作中遇到的各种库的问题总结下，今天想说的是：muduo作者陈硕提到的**“避免使用虚函数作为库的接口”**。

<!--more-->

### 1. 使用虚函数作为动态库的接口可能的问题

#### 1.1. 生成第一个版本的动态库

我们看下使用虚函数作为动态库的接口时可能发生什么。

动态库libfoo.so提供了foo.h作为接口文件

```cpp
#ifndef _SHARED_LIB_FOO_H
#define _SHARED_LIB_FOO_H

class Foo {
public:
    virtual void foo();
    virtual void bar();
};//Foo
#endif  //_SHARED_LIB_FOO_H
```

对应的，我们实现了`foo/bar`两个接口函数

```
#include "foo.h"
#include <stdio.h>

void Foo::foo() { printf("Foo::foo\n"); }
void Foo::bar() { printf("Foo::bar\n"); }
```

生成第一个版本的动态库

```
libfoo.so:shared_lib/v1/foo.cpp
        g++ -fPIC -o shared_lib/v1/foo.o -c $<
        g++ -shared -o $@ shared_lib/v1/foo.o
```


#### 1.2. 在main.cpp里使用

main.cpp使用了libfoo.so，调用了`Foo::foo Foo::bar`：

```
#include "foo.h"

int main() {
    Foo* foo = new Foo();
    foo->foo();
    foo->bar();
    delete foo;

    return 0;
}
```

对应的makefile

```
main:main.cpp
        g++ -o main main.cpp -L . -lfoo -I shared_lib/
```

输出数据跟我们预想的一样，v1的动态库非常完美！

```
Foo::foo
Foo::bar
```


#### 1.2. 生成第二个版本的动态库

很快，升级的需求来了，我们需要提供一个新的接口成员函数，修改后的`foo.h`我们增加了接口`another_func_in_new_version`：

```
#ifndef _SHARED_LIB_FOO_H
#define _SHARED_LIB_FOO_H

class Foo {
public:
    virtual void foo();
    virtual void another_func_in_new_version();
    virtual void bar();
};//Foo
#endif  //_SHARED_LIB_FOO_H
```

对应在`foo.cpp`里增加实现

```
void Foo::another_func_in_new_version() { printf("Foo::another_func_in_new_version\n"); }
```

#### 1.3. 使用升级后的so

因为只是增加了一个接口，我们可能认为并不存在二进制的兼容性问题，不需要发布一个2.0版本。直接替换main所使用的的libfoo.so。

```
$ ./main 
./main: Symbol `_ZTV3Foo' has different size in shared object, consider re-linking
Foo::foo
Foo::another_func_in_new_version
```

问题出现了：`foo->bar()`执行的看起来是我们新加入的函数。合理么？明显不合理，可是我们只是增加了一个接口，就需要重新编译main？那么动态库的便利何在？

如果你完全了解为什么会这样，那么可以忽略文章接下来的内容了。

### 2. 为什么要避免使用虚函数作为动态库的接口

简言之，虚函数的调用是在运行时通过vtable的offset寻址的。这也是上面执行了`another_func_in_new_version`的原因。

网上虚函数文章的介绍很多，关于vtable 陈浩有篇文章详细而且准确。

本文想从另外一个角度介绍下vtable和虚函数的运行机制。

用到的示例代码很简单，我们定义了`Base`基类以及三个虚函数`~Base foo bar`

```
#include <stdio.h>

class Base {
public:
    virtual ~Base() {}

    virtual void foo() {}
    virtual void bar() {}
};//Base

int main() {
    Base* pBase = new Base();
    pBase->foo();
    delete pBase;

    return 0;
}
```

#### 2.1. 符号表里的vtable地址略有不同

编译后我们gdb看下`Base`的vtable地址

```
Breakpoint 1, main () at test_vptr.cpp:13
13          pBase->foo();
(gdb) x /xg pBase
0x601010:       0x0000000000400950
(gdb) x /4xg 0x0000000000400950
0x400950 <_ZTV4Base+16>:        0x00000000004007ca      0x00000000004007f8
0x400960 <_ZTV4Base+32>:        0x000000000040081e      0x0000000000400828
(gdb) info symbol 0x00000000004007ca
Base::~Base() in section .text of /home/users/yingshin/Training/test/test_vptr
(gdb) info symbol 0x00000000004007f8
Base::~Base() in section .text of /home/users/yingshin/Training/test/test_vptr
(gdb) info symbol 0x000000000040081e
Base::foo() in section .text of /home/users/yingshin/Training/test/test_vptr
(gdb) info symbol 0x0000000000400828
Base::bar() in section .text of /home/users/yingshin/Training/test/test_vptr
```

跟上面提到的那篇非常著名的文章一样，我们取到对象的首地址内容作为数组指针，取到后面的函数入口地址，通过`info symbol`得到对应的函数名（可能你看到有两个`Base::~Base`很奇怪，本文暂时不解释原因，我们重点关注下两点：

1. vtable的地址是`0x0000000000400950`  
2. `~Base foo bar`的入口地址确实在vtable数组里  

这里有一个与本文主题无关但是很有意思的一个现象，我们继续看下符号表里`Base`的vtable地址

```
$ nm test_vptr | c++filt  | grep vtable | grep Base
0000000000400940 V vtable for Base
```

很奇怪，不是么？运行时vtable地址和符号表里的并不一致，符号表里的要小16个字节（不同机器环境可能不同）。

探索下这16个字节到底写入了什么？用`objdump -C -d -j .rodata test_vptr`看下`rodata`段


```
0000000000400940 <vtable for Base>:
|   ...
  400948:|  80 09 40 00 00 00 00 00 ca 07 40 00 00 00 00 00     ..@.......@.....
  400958:|  f8 07 40 00 00 00 00 00 1e 08 40 00 00 00 00 00     ..@.......@.....
  400968:|  28 08 40 00 00 00 00 00                             (.@..... 
```

看到前面被写入了`400980`这个地址，继续看下

```
0000000000400980 <typeinfo for Base>:
  400980:|  10 0e 60 00 00 00 00 00 70 09 40 00 00 00 00 00     ..`.....p.@.....
```

这下变得明了了，前面的字节存储的是`typeinfo for Base`，实际上：

当指定编译器打开RTTI开关时，vtable中的第1个指针指向的是一个typeinfo的结构，每个类只产生一个typeinfo结构的实例。当程序调用typeid()来获取类的信息时，实际上就是通过vtable中的第1个指针获得了typeinfo。更详细建议阅读c++对象模型一书。

#### 2.2. 虚函数的动态决议

vtable的设计使得虚函数的动态决议得以实现，虚函数的寻址和普通函数的寻址有什么区别？我们在上例增加一个普通函数看下，对应的diff

```
9d8
<     void non_virtual_foo() {}
14,15c13
<     pBase->bar();
<     pBase->non_virtual_foo();
---
>     pBase->foo();
```

贴下main函数里对应的汇编

```
int main() {
  400760:   55                      push   %rbp
  400761:   48 89 e5                mov    %rsp,%rbp
  400764:   53                      push   %rbx
  400765:   48 83 ec 18             sub    $0x18,%rsp
    Base* pBase = new Base();
  400769:   bf 08 00 00 00          mov    $0x8,%edi
  40076e:   e8 9d fe ff ff          callq  400610 <_Znwm@plt>
  400773:   48 89 c3                mov    %rax,%rbx
  400776:   48 c7 03 00 00 00 00    movq   $0x0,(%rbx)
  40077d:   48 89 df                mov    %rbx,%rdi
  400780:   e8 c3 00 00 00          callq  400848 <_ZN4BaseC1Ev>
  400785:   48 89 5d e8             mov    %rbx,-0x18(%rbp) //pBase的值存储到-0x18(%rbp)
    pBase->bar();
  400789:   48 8b 45 e8             mov    -0x18(%rbp),%rax //pBase的值存储到rax
  40078d:   48 8b 00                mov    (%rax),%rax //pBase指向的内存值存储到rax，也就是vtable的起始位置
  400790:   48 83 c0 18             add    $0x18,%rax //rax += 0x18，移动vtable的指针
  400794:   48 8b 00                mov    (%rax),%rax //rax指向的内存地址存储到rax，也就是取到对应的虚函数入口地址
  400797:   48 8b 55 e8             mov    -0x18(%rbp),%rdx //pBase的值存储到rdx
  40079b:   48 89 d7                mov    %rdx,%rdi //pBase的值存储到rdi，作为接下来调用函数的第一个参数
  40079e:   ff d0                   callq  *%rax //调用rax指向的函数地址，也就是对应的虚函数
    pBase->non_virtual_foo();
  4007a0:   48 8b 45 e8             mov    -0x18(%rbp),%rax //pBase的值存储到rdx
  4007a4:   48 89 c7                mov    %rax,%rdi //pBase的值存储到rdi，作为接下来调用函数的第一个参数
  4007a7:   e8 92 00 00 00          callq  40083e <_ZN4Base15non_virtual_fooEv> //根据名字查找直接调用Base::non_virtual_foo
```

可以看到`pBase->bar`与`pBase->non_virtual_foo`的入口寻址方式是不一样的，虚函数采用vtable的偏移量，而普通函数则通过名字查找的方式。

`pBase->bar`执行时，先取到`pBase`指向的内存起始地址到寄存器`rax`，接着取vtable的实际地址到`rax`，然后通过偏移量0x18的内存数据确定了`pBase->bar`的入口地址。而0x18确实是`bar`在vtable里的偏移量。

```
(gdb) p pBase
$2 = (Base *) 0x601010
(gdb) x /xg pBase                   
0x601010:       0x0000000000400970
(gdb) x /4xg 0x0000000000400970     
0x400970 <_ZTV4Base+16>:        0x00000000004007d6      0x0000000000400804
0x400980 <_ZTV4Base+32>:        0x000000000040082a      0x0000000000400834
(gdb) info symbol 0x0000000000400834
Base::bar() in section .text of /home/users/yingshin/Training/test/test_vptr
```

#### 2.3. 虚函数为什么要用偏移量的方式来寻址

到上一节我们已经逐渐分析出了答案，可能还有朋友像我之前刚了解到一样有个疑问：从汇编上看偏移量的方式明显比普通函数查找要慢，为什么虚函数选用了这种方式呢？

可能原因的由来要追溯C++的历史，这里只分享下我思考的原因。

如果我们从`Base`继承了一个子类`Derived`，没有override任何虚函数。子类`Derived`同样有自己的vtable，每一个子对象都有一个vptr指向该vtable。

但是请注意，`Derived`的vtable存储的函数入口地址都是`Base`里的虚函数，也就是我们从接触c++多态就了解到的：如果子类没有实现该虚函数，则调用父类的虚函数。C++编译的时候出于elf文件大小的考虑，自然不会再去生成一份相同代码的`Derived::foo Derived::bar`了，既然这些函数都没有名字，那么当然是需要用offset的方式来查找入口了。

感兴趣的朋友可以分析下`Derived`的vtable内容。


### 3. 那么你应该怎么做

我想，到这里已经说明白了：由于虚函数通过vtable的偏移量来寻址，因此当升级动态库的时候如果修改了之前的虚函数声明的顺序，或者新增了虚函数（vtable按照基类声明的顺序排列），都可能导致查找到的函数入口地址错误。同样不要试图在末尾添加新的接口的方式，因为你发布的类可能已经被继承。

关于如何设计动态库的接口，陈硕的博客做了详细的介绍。本文避免拾人牙慧不再赘述。`pimpl`的建议非常通用，可以参考[Qt](https://github.com/qt)在接口类里大量使用了这种方式，比如[QWidget](https://github.com/qt/qtbase/blob/dev/src/widgets/kernel/qwidget.h)的接口调用最后都会由`QWidgetData *data;`来实现。
