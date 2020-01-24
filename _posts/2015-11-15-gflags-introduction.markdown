---
title: "参数解析gflags介绍"
date: 2015-11-15 00:22:58
excerpt: "参数解析gflags介绍"
tags: [gflags]
---

gflags是google一个开源的处理命令行参数的库，相比getopt，更加容易使用。  
gflags里参数的定义可以分散在各个源文件处，而不是只能在main文件，使得使用更加灵活，复用性更强。  

<!--more-->
当然，如果两个源文件定义了相同的flag，链接时会报重复定义的错误。  

看个简单的例子:   

```cpp
#include <iostream>
#include <gflags/gflags.h>

DEFINE_bool(big_menu, true, "Include 'advanced' options in the menu listing");
DEFINE_string(languages, "english,french,german",
        "comma-separated list of languages to offer in the 'lang' menu");

int main(int argc, char* argv[])
{
    google::ParseCommandLineFlags(&argc, &argv, true);
    std::cout << FLAGS_big_menu << std::endl;
    std::cout << FLAGS_languages << std::endl;
    return 0;
}
```

上述代码包含了引用头文件、定义、解析、使用flags。

输出如下：  
```
1
english,french,german
```

### 1. 如何定义

flag的定义方法如下：  

```cpp
#include <gflags/gflags.h>

DEFINE_bool(big_menu, true, "Include 'advanced' options in the menu listing");
DEFINE_string(languages, "english,french,german",
             "comma-separated list of languages to offer in the 'lang' menu");
```

定义flag一共6个宏，分别对应6种类型：  

```cpp
DEFINE_bool: boolean
DEFINE_int32: 32-bit integer
DEFINE_int64: 64-bit integer
DEFINE_uint64: unsigned 64-bit integer
DEFINE_double: double
DEFINE_string: C++string
```

注意DEFINE_xxx函数的3个参数都是必须的。

可以定义到某个 namespace 下，使用时也需要带着 namespace 前缀.

### 2. 查看程序支持了哪些flags

例如对上述flags的定义，-help输出如下：  

```cpp
  Flags from flags_help.cpp:
    -big_menu (Include 'advanced' options in the menu listing) type: bool
      default: true
    -languages (comma-separated list of languages to offer in the 'lang' menu)
      type: string default: "english,french,german"
```

### 3. 如何使用

在命令行里指定，例如`./flags_sample -big_menu=0 -languages="english"`  
使用时，对应的变量名为`FLAGS_xxx`。  
如果不想在命令行里指定，也可以使用-flagfile=文件名的形式。  
如果使用时不想在main文件里定义flag，例如需要在flag1.cpp flags.cpp里定义,可以分别声明和定义:  
在flags1.cpp flags2.cpp里分别定义各自的flag，然后在flag.h声明，需要使用的文件直接`include flags.h`就可以了。  
声明的函数如下：  

```cpp
   DECLARE_bool(big_menu);
```

### 4. 如何解析

`gflags::ParseCommandLineFlags(&argc, &argv, remove_flags)`可以帮助解析出相应的flags  
__argc argv__: main中的对应参数  
注意这里的参数为[in | out]  
__remove\_flags__: 若设置为true，表示解析后将flag以及flag对应的值从argv中删除，并相应的修改argc，即最后存放的是不包含flag的参数。如果设置为false，则仅对参数进行重排，标志位参数放在最前面。   
返回值：文档中说明如下，   

> ParseCommandLineFlags returns the index into argv that holds the first commandline argument: that is, the index past the last flag.  

我觉得是返回处理后的argv第一个非flag值的下标  

也可以在命令行传入`--flagfile`或者在程序里设置`flagfile`以解析文件中的flags。

```cpp
google::SetCommandLineOption("flagfile", "gflags_sample.flags");
```

`FLAGS_flagfile`更新后，会自动重新读取该文件并更新文件里的gflags。

另外看过厂里很多代码使用`ReadFromFlagsFile`接口，不过该接口已经标明`DEPRECATED`，不建议使用。  

```cpp
// These let you manually implement --flagfile functionality.
// DEPRECATED.
extern bool AppendFlagsIntoFile(const std::string& filename, const char* prog_name);
extern bool ReadFromFlagsFile(const std::string& filename,
                              const char* prog_name,
                              bool errors_are_fatal);   // uses SET_FLAGS_VALUE
```

这几种方法可以同时使用以起到reload的效果，后者覆盖前者，如果后面调用的方法没有定义该flag，那么不影响前面方法已经解析出的value，类似于merge的效果。

### 5. 检查有效性

gflags提供了一个检查传入flag值是否有效的功能，只要定义检测函数，并且注册就可以了。  
检测函数以及注册方式的例子：  

```cpp
static bool ValidatePort(const char* flagname, int32 value) {
   if (value > 0 && value < 32768)   // value is ok
     return true;
   printf("Invalid value for --%s: %d\n", flagname, (int)value);
   return false;
}
DEFINE_int32(port, 0, "What port to listen on");
static const bool port_dummy = RegisterFlagValidator(&FLAGS_port, &ValidatePort);
```

如果注册成功，regist函数返回值为ture。否则返回false，注册失败一般是一下两种原因：
1. 第一个参数不是flag
2. 该flag已经注册过

写了一个完整的[例子](https://gist.github.com/yingshin/35a17cc4a6631002d3e0)放在了gist上。

### 6. 判断一个FLAG是否被设置

使用`GetCommandLineFlagInfo`即可  
例如判断`portno`是否设置：  

```cpp
    google::CommandLineFlagInfo info;
    if (GetCommandLineFlagInfo("portno", &info) && info.is_default) {
        std::cout << "port is not set." << std::endl;
    } else {
        std::cout << "port is set." << std::endl;
    }
```

注意这里**不是简单比较flag值是否与默认值相同**，如果设置了flag=默认值，也会输出"port is set"。

### 7. 在程序里设置FLAG

实际项目里，我们使用gflag替代了传统conf配置，其中有一个需求是配置可以动态reload的。简单点通过手动修改的方式： `FLAGS_protno = 9999`。  
比较合理的是使用`SetCommandLineOption`，函数原型为  

```cpp
extern std::string SetCommandLineOption(const char* name, const char* value);
```

注意bool int类型都使用字符串的方式修改，例如：

```cpp
google::SetCommandLineOption("bvar_dump", "true")
google::SetCommandLineOption("portno", "9999")
```

成功返回"portno set to 9999"，失败则返回空字符串。

与此类似的是读取flag的接口：

```cpp
extern bool GetCommandLineOption(const char* name, std::string* OUTPUT)
```
OUTPUT填充了对应的设置的值，如果一个flag未设置，那么OUTPUT将填充默认值。无论flag是否设置均认为获取成功返回true，
如果name是一个未定义的flag，则认为获取失败返回false。

读写其实都可以使用`if (FLAGS_foo); FLAGS_Foo = bar`的形式，但是如果需要线程安全的调用，使用`GetCommandLineOption/SetCommandLineOption`接口。

### 8. version与help

一般我们的程序都需要-version提供版本信息，-help提供Usage。  
可以使用`SetVersionString` 和 `SetUsageMessage` 来实现。  

### 9. 遍历所有的flags

使用`extern void GetAllFlags(std::vector<CommandLineFlagInfo>* OUTPUT)`接口。  

更多的使用接口，可以直接查看gflags/gflags.h。

### 10. 保留gflag

gflags源码里有些保留的flag，不要重复定义

```cpp
// Special flags, type 1: the 'recursive' flags.  They set another flag's val.
DEFINE_string(flagfile,   "", "load flags from file");
DEFINE_string(fromenv,    "", "set flags from the environment"
                              " [use 'export FLAGS_flag1=value']");
DEFINE_string(tryfromenv, "", "set flags from the environment if present");

// Special flags, type 2: the 'parsing' flags.  They modify how we parse.
DEFINE_string(undefok, "", "comma-separated list of flag names that it is okay to specify "
                           "on the command line even if the program does not define a flag "
                           "with that name.  IMPORTANT: flags in this list that have "
                           "arguments MUST use the flag=value format");
```

否则会报错

```
multiple definition of 'fLS::FLAGS_flagfile'
```

### 11. 重复定义

这个问题跟上面类似，不要定义相同的flag名，即使放在不同的namespace。

例如我们定义了

```cpp
namespace foo {
DEFINE_bool(multi_thread_enabled, true, "");
}
namespace bar {
DEFINE_bool(multi_thread_enabled, true, "");
}
```

编译没有问题，但是运行会报错：

```cpp
ERROR: flag 'multi_thread_enabled' was defined more than once
```

或者

```cpp
ERROR: something wrong with flag ...
```

具体可以参考gflags源码：

```cpp
  if (ins.second == false) {   // means the name was already in the map
    if (strcmp(ins.first->second->filename(), flag->filename()) != 0) {
      ReportError(DIE, "ERROR: flag '%s' was defined more than once "
                  "(in files '%s' and '%s').\n",
                  flag->name(),
                  ins.first->second->filename(),
                  flag->filename());
    } else {
      ReportError(DIE, "ERROR: something wrong with flag '%s' in file '%s'.  "
                  "One possibility: file '%s' is being linked both statically "
                  "and dynamically into this executable.\n",
                  flag->name(),
                  flag->filename(), flag->filename());
    }
  }
```

### 12. 参考资料

[https://gflags.github.io/gflags/](https://gflags.github.io/gflags/)
