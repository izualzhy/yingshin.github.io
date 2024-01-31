---
title: glog源码解析四：如何获取函数的调用栈
date: 2017-06-24 12:15:36
excerpt: "glog源码解析"
tags: glog
---

glog在fatal时会提前打印函数的调用栈到日志，平时工作大部分时候我们关心调用栈基本都是通过gdb core查看。今天介绍下如何在正常情况下获取函数的调用栈。

<!--more-->

先看下我们的测试用的代码

```
void foo_3() {
    get_backtrace_by_backtrace();
}

void foo_2() {
    foo_3();
}

void foo_1() {
    foo_2();
}

void foo() {
    foo_1();
}

int main()
{
    foo();
    return 0;
}
```

其中`get_backtrace_by_backtrace`是我们获取栈的代码实现。调用顺序`main->foo->foo_1->foo_2->foo_3->get_backtrace+by_backtrace`。

## 1. 通过glibc的`backtrace`函数

```
void get_backtrace_by_backtrace() {
    const int backtrace_max_size = 100;
    void *buffer[backtrace_max_size];

    int buffer_size = backtrace(buffer, backtrace_max_size);

    char **symbols = backtrace_symbols(buffer, buffer_size);
    for (int i = 0; i < buffer_size; ++i) {
        printf("%p:%s\n", buffer[i], symbols[i]);
    }
    free(symbols);
}
```

其中`backtrace backtrace_symbols`都是库函数，先获取到地址，然后解析出对应的symbol，编译命令如下

```
g++  -g -Wall -Werror -std=c++11 -o test_backtrace test_backtrace.cpp -rdynamic
```

注意`-rdynamic`是必须的，否则无法得到符号表的名字，执行下`test_backtrace`

```
0x400a46:./test_backtrace(_Z26get_backtrace_by_backtracev+0x26) [0x400a46]
0x400ac3:./test_backtrace(_Z5foo_3v+0x9) [0x400ac3]
0x400ace:./test_backtrace(_Z5foo_2v+0x9) [0x400ace]
0x400ad9:./test_backtrace(_Z5foo_1v+0x9) [0x400ad9]
0x400ae4:./test_backtrace(_Z3foov+0x9) [0x400ae4]
0x400aef:./test_backtrace(main+0x9) [0x400aef]
0x7f37d5ffebd5:/top/lib/libc.so.6(__libc_start_main+0xf5) [0x7f37d5ffebd5]
0x400909:./test_backtrace() [0x400909]
```

`c++filt`下符号会更加易读

```
0x400a46:./test_backtrace(get_backtrace_by_backtrace()+0x26) [0x400a46]
0x400ac3:./test_backtrace(foo_3()+0x9) [0x400ac3]
0x400ace:./test_backtrace(foo_2()+0x9) [0x400ace]
0x400ad9:./test_backtrace(foo_1()+0x9) [0x400ad9]
0x400ae4:./test_backtrace(foo()+0x9) [0x400ae4]
0x400aef:./test_backtrace(main+0x9) [0x400aef]
0x7f9d965b9bd5:/top/lib/libc.so.6(__libc_start_main+0xf5) [0x7f9d965b9bd5]
0x400909:./test_backtrace() [0x400909]
```

从这个例子可以看到获取函数调用栈，分了两步：第一步获取到地址，第二步解析成符号。

解析成符号这一步也可以使用`dladdr`函数

```
    for (int i = 0; i < buffer_size; ++i) {
        Dl_info dlinfo;
        if (dladdr(buffer[i], &dlinfo) != 0) {
            printf("%p:%s\n", buffer[i], dlinfo.dli_sname);
        }
    }
```

注意编译时需要增加一个选项`-ldl`

输出的调用栈如下：

```
0x400a16:_Z26get_backtrace_by_backtracev
0x400a87:_Z5foo_3v
0x400a92:_Z5foo_2v
0x400a9d:_Z5foo_1v
0x400aa8:_Z3foov
0x400ab3:main
0x7f1e01d54bd5:__libc_start_main
0x4008d9:(null)
```

## 2. 通过libunwind

libunwind提供了一系列接口回溯栈，需要编译libunwind.a

```
void get_backtrace_by_unwind() {
    unw_cursor_t    cursor;
    unw_context_t   context;

    unw_getcontext(&context);
    unw_init_local(&cursor, &context);

    while (unw_step(&cursor) > 0) {
        unw_word_t  offset, pc;
        char symbols[64] = {0};

        unw_get_reg(&cursor, UNW_REG_IP, &pc);
        unw_get_proc_name(&cursor, symbols, sizeof(symbols), &offset);
        printf ("%lu : (%s+0x%lu) [%lu]\n", pc, symbols, offset, pc);
    }
}
```

编译命令如下：

```
g++  -g -Wall -Werror -std=c++11 -o test_backtrace test_backtrace.cpp -I/home/users/yingshin/local/include /home/users/yingshin/local/lib/libunwind-x86_64.a /home/users/yingshin/local/lib/libunwind.a
```

输出：

```
4199016 : (_Z5foo_3v+0x9) [4199016]
4199027 : (_Z5foo_2v+0x9) [4199027]
4199038 : (_Z5foo_1v+0x9) [4199038]
4199049 : (_Z3foov+0x9) [4199049]
4199060 : (main+0x9) [4199060]
139986043538389 : (__libc_start_main+0x245) [139986043538389]
4198521 : (_start+0x41) [4198521]
```

## 3. glog里的做法

回到题目，看下glog里的做法。glog里要复杂一些，包括各平台的兼容，符号的解析，使得输出更加易读，例如

```
*** Check failure stack trace: ***
    @           0x40725a  google::LogMessage::Fail()
    @           0x408f9f  google::LogMessage::SendToLog()
    @           0x406e98  google::LogMessage::Flush()
    @           0x4098be  google::LogMessageFatal::~LogMessageFatal()
    @           0x405c1d  main
    @     0x7fc39d24abd5  __libc_start_main
    @           0x405919  (unknown)
```

glog初始化时会安装一个failure function，默认为`g_logging_fail_func = DumpStackTraceAndExit`，负责导出栈数据的为`DumpStackTrace`函数。

```
// Dump current stack trace as directed by writerfn
static void DumpStackTrace(int skip_count, DebugWriter *writerfn, void *arg) {
  // Print stack trace
  void* stack[32];
  int depth = GetStackTrace(stack, ARRAYSIZE(stack), skip_count+1);
  for (int i = 0; i < depth; i++) {
#if defined(HAVE_SYMBOLIZE)
    if (FLAGS_symbolize_stacktrace) {
      DumpPCAndSymbol(writerfn, arg, stack[i], "    ");
    } else {
      DumpPC(writerfn, arg, stack[i], "    ");
    }
#else
    DumpPC(writerfn, arg, stack[i], "    ");
#endif
  }
}
```

可以看到也是先Get到trace，然后再dump成符号。

glog一共实现了5个`GetStackTrace`，位于各个`stacktrace_xxx`文件，有的实现跟我们上面介绍的例子一致，有的则是直接回溯`rbp rsp`的形式。

`DumpPCAndSymbol`通过`Symbolize`完成，位于`symbolize.cc`，也是实现了类似`c++filt`的功能。

`chromium`源码里也是类似的做法，感兴趣的同学可以看下`base/debug/stack_trace_posix.cc`，有的文件名甚至相同个，比如`base/third_party/symbolize/symbolize.h`。


