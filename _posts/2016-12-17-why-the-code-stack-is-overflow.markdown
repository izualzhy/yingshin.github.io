---
title: 追core笔记之五：如何查看一个corrupt stack的core
date: 2016-12-17 16:13:46
excerpt: "追core笔记之五：如何查看一个corrupt stack的core"
tags: core
---

接触c以来有很多好奇的问题，其中一类是关于栈的。比如：栈上存储了哪些数据？函数参数怎么传递的？返回值怎么传出去的？从一个函数是怎么跳转到另外一个函数的？为何gdb可以看到函数的调用栈？为何有些栈的信息会乱？

如果要讲清楚core栈的信息为何有很多问号，想了下觉得应该先从如何确定函数的调用栈来讲。

注：本文会花大部分篇幅介绍栈上存储了什么，如果制造一个corrupt stack的core，解决方案根据经验看往往并不固定，希望对读者有用。

<!--more-->

### 1. call-trace是如何确定的

我们知道栈是向下增长的，更具体些，栈顶由寄存器`esp`进行定位。压栈操作使栈顶地址减少，弹出操作使栈顶地址增大。

`esp`寄存器始终指向栈顶，而`ebp`则指向一个固定的位置，又称为帧指针（Frame Pointer）。我们先通过gdb看一个例子，观察下函数调用过程中`ebp esp`的变化，栈上数据发生了什么样的改变。

#### 1.1. 函数调用时栈的变化

函数调用时，栈上有一些数据是我们预先定义的，比如参数、各种指令，有些则是编译器为了寻址、回溯添加进去的。

先贴下示例用的代码，使用gcc4.8.2在64位环境下编译，所以接下来使用`rbp rsp`来描述。

函数调用关系为：`main->bar->foo`

```
void foo(const char* arg) {
    char buf[16] = {0};
    strcpy(buf, arg);
    printf("foo arg:%s\n", buf);
}

void bar(int argc, const char* arg) {
    printf("bar argc:%d\n", argc);
    foo(arg);
}

int main(int argc, char** argv) {
    if (argc < 2) {
        printf("argument count < 2.\n");
        return -1;
    }

    bar(argc, argv[1]);

    return 0;
}
```

那么接下来，我们使用gdb看下调用`bar`后的栈的变化（注意由于编译环境的不同，对应的符号地址可能会略有变化）

传入的参数为"helloworld"，我们在`bar`处设置下断点：

```
(gdb) set args helloworld
(gdb) disass main
Dump of assembler code for function main(int, char**):
   0x0000000000400778 <+0>:     push   %rbp
   0x0000000000400779 <+1>:     mov    %rsp,%rbp
   0x000000000040077c <+4>:     sub    $0x10,%rsp
   0x0000000000400780 <+8>:     mov    %edi,-0x4(%rbp)
   0x0000000000400783 <+11>:    mov    %rsi,-0x10(%rbp)
   0x0000000000400787 <+15>:    cmpl   $0x1,-0x4(%rbp)
   0x000000000040078b <+19>:    jg     0x40079e <main(int, char**)+38>
   0x000000000040078d <+21>:    mov    $0x4008a5,%edi
   0x0000000000400792 <+26>:    callq  0x400590 <puts@plt>
   0x0000000000400797 <+31>:    mov    $0xffffffff,%eax
   0x000000000040079c <+36>:    jmp    0x4007bb <main(int, char**)+67>
   0x000000000040079e <+38>:    mov    -0x10(%rbp),%rax
   0x00000000004007a2 <+42>:    add    $0x8,%rax
   0x00000000004007a6 <+46>:    mov    (%rax),%rdx
   0x00000000004007a9 <+49>:    mov    -0x4(%rbp),%eax
   0x00000000004007ac <+52>:    mov    %rdx,%rsi
   0x00000000004007af <+55>:    mov    %eax,%edi
   0x00000000004007b1 <+57>:    callq  0x400747 <bar(int, char const*)>
   0x00000000004007b6 <+62>:    mov    $0x0,%eax
   0x00000000004007bb <+67>:    leaveq 
   0x00000000004007bc <+68>:    retq   
End of assembler dump.
(gdb) b *0x00000000004007b1
Breakpoint 1 at 0x4007b1: file test_test.cpp, line 21.
(gdb) i r rbp rsp
rbp            0x7fffffffdd60   0x7fffffffdd60
rsp            0x7fffffffdd50   0x7fffffffdd50
```

注意我们的断点设置了地址，而没有使用`b 21`设置，这样可以跳过设置参数的汇编指令，也就是`40079e`这几条。

而使用函数名、行号等作为断点时这几条汇编指令都不会被跳过去：

类似需要注意的还有gdb自动帮我们跳过去的代码，比如接下来`si`调用`bar`：

```
(gdb) si
bar (argc=0, arg=0x7ffff7ba55f0 <vtable for (anonymous namespace)::future_error_category+16> " 677") at test_test.cpp:10
10      void bar(int argc, const char* arg) {
(gdb) disass
Dump of assembler code for function bar(int, char const*):
=> 0x0000000000400747 <+0>:     push   %rbp
   0x0000000000400748 <+1>:     mov    %rsp,%rbp
   0x000000000040074b <+4>:     sub    $0x10,%rsp
   0x000000000040074f <+8>:     mov    %edi,-0x4(%rbp)
   0x0000000000400752 <+11>:    mov    %rsi,-0x10(%rbp)
   0x0000000000400756 <+15>:    mov    -0x4(%rbp),%eax
   0x0000000000400759 <+18>:    mov    %eax,%esi
   0x000000000040075b <+20>:    mov    $0x400898,%edi
   0x0000000000400760 <+25>:    mov    $0x0,%eax
   0x0000000000400765 <+30>:    callq  0x400570 <printf@plt>
   0x000000000040076a <+35>:    mov    -0x10(%rbp),%rax
   0x000000000040076e <+39>:    mov    %rax,%rdi
   0x0000000000400771 <+42>:    callq  0x400700 <foo(char const*)>
   0x0000000000400776 <+47>:    leaveq 
   0x0000000000400777 <+48>:    retq 
```

如果使用`b bar`，则断点停在`0x0000000000400756`指令处，也就是已经运行了以下这些指令：

```
void bar(int argc, const char* arg)
  400747:       55                      push   %rbp
  400748:       48 89 e5                mov    %rsp,%rbp
  40074b:       48 83 ec 10             sub    $0x10,%rsp
  40074f:       89 7d fc                mov    %edi,-0x4(%rbp)
  400752:       48 89 75 f0             mov    %rsi,-0x10(%rbp)
    printf("bar argc:%d\n", argc);
```

言归正传，看下运行到`bar`汇编指令第一行后`rsp rbp`寄存器变化

```
(gdb) i r rbp rsp
rbp            0x7fffffffdd60   0x7fffffffdd60
rsp            0x7fffffffdd48   0x7fffffffdd48
```

`callq`指令运行后，可以看到栈顶指针`rsp`减少了8字节的空间，我们看下这个8字节存了什么?

```
(gdb) x /xg $rsp
0x7fffffffdd48: 0x00000000004007b6
(gdb) info symbol 0x00000000004007b6
main + 62 in section .text of /home/users/y/Training/test/test_test
```

从前面`main`的指令，也可以看到**这里存储了`bar`函数返回后需要调用的`main`下一条指令**，这是堆栈帧(Stack Frame)的一部分。

继续运行下一条指令，看下对`rbp rsp`的影响：

```
(gdb) ni
0x0000000000400748      10      void bar(int argc, const char* arg) {
(gdb) i r rbp rsp
rbp            0x7fffffffdd60   0x7fffffffdd60
rsp            0x7fffffffdd40   0x7fffffffdd40
(gdb) x /xg $rsp
0x7fffffffdd40: 0x00007fffffffdd60
```

看到`push   %rbp`将上个rbp存储的内容写到栈上。

继续运行看下一条指令：

```
(gdb) ni
0x000000000040074b      10      void bar(int argc, const char* arg) {
(gdb) i r rbp rsp
rbp            0x7fffffffdd40   0x7fffffffdd40
rsp            0x7fffffffdd40   0x7fffffffdd40
```

`mov %rsp,%rbp`设置rbp存储了跟rsp相同的一段地址值`0x7fffffffdd40`，对应的内容从上条指令可以看到是上一个rbp的值。

接下来的几个指令，就是增长栈空间，将`rsi edi`存储的值，也就是参数值放到栈上，然后开始真正函数实现的调用。
接下来跟`main`调用`bar`类似，`bar`调用`foo`。

我们先贴一下`foo`前面几行汇编代码：

```
void foo(const char* arg)
  400700:       55                      push   %rbp
  400701:       48 89 e5                mov    %rsp,%rbp
  400704:       48 83 ec 20             sub    $0x20,%rsp
  400708:       48 89 7d e8             mov    %rdi,-0x18(%rbp)
```

为了更直观的了解下`rsp rbp`的变化流程，省去gdb单步调试的部分，画了一张图描述了下几个指令执行后栈上的变化

![call-trace-rsp-rbp](/assets/images/call-trace-rsp-rbp.png)

关于图形格式的一些解释：

1. 红色为标题行，表示调用对应的assemble code后的状态。下面是当时部分栈上的状态。  
2. 由于使用的是64位系统，因此每个框表示8个字节。  
3. 绿色、蓝色分别为`rbp` `rsp`所存储的8字节栈上空间的首地址。  
4. 黄色表示此时`rbp` `rsp`存储了相同的值，指向了同一块内存  
5. 框内内容为该8字节内存的地址，如有必要，还有些其他说明。  

从这张图我们可以清晰的看到每条指令执行后栈上的变化，比如`bar`函数调用了`move %rbp %rsp`后`rbp`跟`rsp`存储了同一块内存地址。那么栈上为什么会写入了这些数据？`rbp`有什么作用？接下来想从一个最基本的问题“栈是什么”来解释下，并分析下上面这张图为什么会填充了一些诸如`返回指令`、`prev rbp`这样的内容。

#### 1.2 栈是什么

寄存器的操作导致了栈的变化，上图中我们可以看到在一级级的函数调用时，栈一直在向下增长。除了我们需要的指令、参数、变量以外，每次函数调用时栈上的数据还会发生一些自动的变化，比如`push %rbp`。

实际上栈保存了一个函数调用所需要的上下文信息，被称为堆栈帧(Stack Frame)或者活动记录（Activate Record）。

上图可以清晰的看到`rsp`始终指向了栈顶，而`rbp`的作用还有些不够明显。先看下i386函数的标准开头

```
push rbp             //%rbp压入栈中，称为prev rbp, rsp += 8
mov rbp rsp          //%rbp = %rsp，也就是当前rbp存储了prev rbp的地址
```

函数调用开始后，本函数内`rbp`不再发生变化，始终存储了`prev rbp`的地址。这也是`rbp`寄存器又被称为帧指针(Frame Pointer)的原因，下图绘制了当前栈上`rbp`的链式关系，可以直观的看到对应链表关系，当函数调用深度增加，这个回溯的链接不会变。

![chains-of-rbp](/assets/images/chains-of-rbp.png)

那么问题来了，最前面的`rbp`存储了什么，图里也给出了答案：0，从另一方面显示了`main`函数特殊的地位。

实际上栈上保存的数据比示例里复杂一些：

+ 函数的返回地址和参数  
+ 临时变量： 包括函数的非静态局部变量以及编译器自动生成的其他临时变量  
+ 保存的上下文： 包括在函数调用前后需要保持不变的寄存器

例如这样：

![active_record.png](/assets/images/active_record.png)

*该照片剪切自《程序员的自我修养》一书*

栈上存储返回后执行的下一条指令是很容易理解的，因为必须要在函数调用后给他下一条指令的入口。那么存储`rbp`的作用是什么呢？前面一直在分析进入函数时栈上数据的变化，现在我们先看下函数退出对栈的影响。

我们还是gdb看下`foo`函数退出时栈上的变化：

```
=> 0x0000000000400745 <+69>:    leaveq 
   0x0000000000400746 <+70>:    retq   
End of assembler dump.
(gdb) i r rbp rsp
rbp            0x7fffffffdd20   0x7fffffffdd20
rsp            0x7fffffffdd00   0x7fffffffdd00
```

这是执行`leaveq`前的寄存器情况，对比前面的图片可以看到`rbp`的值未发生变化。

看下执行`leaveq`后的寄存器变化

```
(gdb) ni
0x0000000000400746      8       }
(gdb) i r rbp rsp
rbp            0x7fffffffdd40   0x7fffffffdd40
rsp            0x7fffffffdd28   0x7fffffffdd28
```

`leaveq`实际上是两条指令：

```
movq %rbp, %rsp      //%rbp的值，赋值到rsp，即rsp指向的内存存储了prev_rbp
popq %rbp            //弹出栈顶的数据:prev_rbp到rbp，即rbp的值
```

可以看到跟函数首部的指令正好相反

```
push rbp             //%rbp压入栈中，称为prev rbp, rsp += 8
mov rbp rsp          //%rbp = %rsp，也就是当前rbp存储了prev rbp的地址
```

`leaveq`执行后：

`rsp`指向了`rbp`的上一条指令，也就是`返回后执行的下一条指令`  
`rbp`则存储了`prev_rbp`的值，直观点的结果如图：

![rbp_on_function_exit.png](/assets/images/rbp_on_function_exit.png)

`retq`将栈顶地址pop到`rip`，也就是下一条要执行的指令，我们验证下

```
(gdb) i r rbp rsp rip
rbp            0x7fffffffdd40   0x7fffffffdd40
rsp            0x7fffffffdd28   0x7fffffffdd28
rip            0x400746 0x400746 <foo(char const*)+70>
(gdb) ni
bar (argc=2, arg=0x7fffffffe124 "helloworld") at test_test.cpp:13
13      }
(gdb) i r rbp rsp rip
rbp            0x7fffffffdd40   0x7fffffffdd40
rsp            0x7fffffffdd30   0x7fffffffdd30
rip            0x400776 0x400776 <bar(int, char const*)+47>
```

可以看到这里`rip`指向了`bar`要执行的下一条指令，`foo`函数功成身退，栈上的控制权重新返回给了`bar`函数。

再看下`rbp` `rsp`的值，是不是有些眼熟？对照下前面的call-trace的图，`bar`调用`foo`前的栈上内容，是不是完全一致？

而这，就是`rbp`最重要的作用，从前面也可以看到，`rbp`是一个链式的结构，不断地回溯，可以找到程序最开始的地方，借助`rbp`，`rsp`才能找到退出函数后的下一条指令。你看，程序都是不忘初心，不是么？这也是`rbp`被称为帧指针(Frame Pointer)的原因。

那么，只有借助`rbp`，`rsp`才能找到退出函数的下一条指令吗？

#### 1.3. fomit-frame-pointer

GCC编译器有一个`--fomit-frame-pointer`可以取消帧指针，即不使用任何帧指针，而是通过`esp`直接计算帧上变量的位置。这么做的好处是可以多出一个`ebp`寄存器供使用，但是坏处却很多，比如帧上寻址速度会变慢，而且没有帧指针之后，无法准确定位函数的调用轨迹(Stack Trace)。

举个例子，`foo`函数加了该编译选项后的汇编如下

```
void bar(int argc, const char* arg) {
  40074c:       48 83 ec 18             sub    $0x18,%rsp
  ... ...
  400779:       48 83 c4 18             add    $0x18,%rsp
  40077d:       c3                      retq
```

可以看到rsp的恢复使用了偏移量的形式

查看[gcc的优化选项](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html)，-o1在不影响debug的情况下会开启该选项。

在使用tcmalloc时，需要注意这个选项，具体可以参考[tcmalloc的主页](https://github.com/gperftools/gperftools/blob/master/INSTALL)。

### 2. core栈为什么会乱

在追查core时，我们会发现很多?的情况。这类core确实难以追查，在接手模块的过程中，我遇到了两类core属于这种情况。

从前面的介绍里，我们知道了栈上的数据可以回溯追踪到函数的调用信息，而gdb之所以能够定位core栈，就是通过`rbp`来区分每个函数的帧，也就可以根据`rbp`前面的字节，即函数返回后调用函数的下条指令，逐步回溯到调用函数名。

同时根据前面的分析，当我们数组写越界时，污染的是高地址的数据，比如定义顺序在前的其他变量，或者`prev_rbp`、返回后的下条指令等。写坏的栈数据很难追查，不过根据前面的分析，我们可以得到一些基本的结论：

1. `rbp`比`rsp`的值要大  
2. `rbp`的前一个地址是调用函数的下一条指令  
3. `rbp`遵循链式结构  

### 3. corrupt stack的实例

我们继续用第一节里的代码看下corrupt stack的core实例

```
$ ./test_test helloworldabcdef
bar argc:2
foo arg:helloworldabcdef
Segmentation fault (core dumped)
```

gdb看下

```
$ gdb test_test core.22278
(gdb) bt
#0  0x00007fff1d18613d in ?? ()
#1  0x726f776f6c6c6568 in ?? ()
#2  0x666564636261646c in ?? ()
#3  0x00007fff1d185100 in ?? ()
#4  0x0000000000400776 in bar (argc=684837, 
    arg=0x206f6f6600020001 <error: Cannot access memory at address 0x206f6f6600020001>) at test_test.cpp:12
Backtrace stopped: previous frame inner to this frame (corrupt stack?)
```

果然，一个没有调用栈的core出现了。

这种core实际修复要复杂的多，这里只是简单介绍下一些基本思路：

```
(gdb) fr 0
#0  0x00007fff1d18613d in ?? ()
(gdb) i r rsp rbp rip
rsp            0x7fff1d185110   0x7fff1d185110
rbp            0x400898 0x400898
rip            0x7fff1d18613d   0x7fff1d18613d
```

可以看到`rip`的地址明显不是一个合法的指令，通过`dmesg`也可以验证这点：

```
$ dmesg | tail -n 1
test_test[3932]: segfault at 7fff1d18613d ip 00007fff1d18613d sp 00007fff1d185110 error 15
```

那么，怎么定位呢？

我们先看下`rbp`的值是否合法

```
(gdb) x /xg 0x400898
0x400898:       0x6367726120726162
(gdb) info symbol 0x6367726120726162
No symbol matches 0x6367726120726162.
(gdb) x /s 0x400898
0x400898:       "bar argc:%d\n"
```

可以看到第一个有用的信息，是在使用字符串"bar argc:%d\n"附近。

```
(gdb) x /16xg $rsp
0x7fff1d185110: 0x726f776f6c6c6568      0x666564636261646c
0x7fff1d185120: 0x00007fff1d185100      0x0000000000400776
0x7fff1d185130: 0x00007fff1d18613d      0x0000000200000000
0x7fff1d185140: 0x00007fff1d185160      0x00000000004007b6
0x7fff1d185150: 0x00007fff1d185248      0x0000000200000000
0x7fff1d185160: 0x0000000000000000      0x00007fb397b10bd5
0x7fff1d185170: 0x0000000000000000      0x00007fff1d185248
0x7fff1d185180: 0x0000000200000000      0x0000000000400778
(gdb) info symbol 0x0000000000400776
bar(int, char const*) + 47 in section .text of /home/users/yingshin/Training/test/test_test
```

这里另一个有用的信息，是最近的函数调用指令为`0x400776`，`bar`在调用`foo`后的下一条指令。

猜测问题是否出在`foo`函数，查看源码，`strcpy`这种很容易写栈溢出的函数，对比"helloworldabcdef"的长度，不难得到答案。这里贴一下`foo`的汇编:

```
  400704:       48 83 ec 20             sub    $0x20,%rsp
  400708:       48 89 7d e8             mov    %rdi,-0x18(%rbp)//参数拷贝到$rbp - 0x18内存开始的位置
    char buf[16] = {0};
  40070c:       48 c7 45 f0 00 00 00    movq   $0x0,-0x10(%rbp)//$rbp - 0x10即buf首地址，即如果buf写超过16个字节的话，就会写乱rbp的内存、甚至调用函数的内存
  400713:       00
  400714:       48 c7 45 f8 00 00 00    movq   $0x0,-0x8(%rbp)
  40071b:       00
    strcpy(buf, arg);
  40071c:       48 8b 55 e8             mov    -0x18(%rbp),%rdx
  400720:       48 8d 45 f0             lea    -0x10(%rbp),%rax
  400724:       48 89 d6                mov    %rdx,%rsi
  400727:       48 89 c7                mov    %rax,%rdi
  40072a:       e8 81 fe ff ff          callq  4005b0 <strcpy@plt>
```

当然这只是一个简化版的例子，实际上我在解决core的过程中花了很多的时间，甚至有运气的成分。

只能根据经验总结一下你要埋这么一个core的方法：

1. `strcpy`等不安全的函数要多用
2. 数组定义的时候不要解释传入的常量大小
3. 多使用一些魔数

当然这样还是太基本了些，大部分情况要传入的参数是不会引起core的，最好是在多线程重入的情况下可能导致参数本身是非预期的。。。算了，不说了，两行泪。。。

当需要解决这种问题时，则反其道而行之。

### 4. 总结

这篇文章其实花了很长的篇幅来介绍栈上数据的布局，然而对于解决方案却没有明确给出来。主要还是这类core表现形式往往不同，出现问题后检查一些容易导致栈溢出的函数会比较好一些。更进一步，在平时写代码时，注意使用`strncpy`代替`strcpy`等等操作，多线程全局变量是否符合预期等等，往往比辛苦追查core要更省时间一些。而对于出现的这种core，从栈上开始分析，前面提到的一些定理，也能分析出蛛丝马迹。
