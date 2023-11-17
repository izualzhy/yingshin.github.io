---
title: shell上的gflags - shflags介绍
date: 2018-07-28 16:17:13
excerpt: "shell上的gflags - shflags介绍"
tags: gflags
---

在处理命令行参数的选择上，很多 C++ 项目里都可以见到 gflags 的使用，之前的[笔记](https://izualzhy.cn/gflags-introduction)也简单介绍过。

在具体工作我们做项目时，各个功能使用合适的语言才是最佳选择，因此执行下来往往是多语言的场景。

例如在 shell 脚本里调用 C++ 编译出的二进制( hadoop-job 使用 hadoop-streaming 时很常见），就会出现这样的场景： shell 脚本使用`getopt`处理命令行参数，二进制使用 gflags 处理，如果两者参数有重复，代码往往会变成这样：

```
# analyse param for C++_compile_bin
while getopts "..." opt
do
    case "$opt" in
        "I")
        input_path=$OPTARG
        ;;
do_sth_with_input_path ${input_path}
...

${C++_compile_bin} --input_path=${input_path}
```

多了一层参数解析的环节，出于统一以及易维护的角度，我们会希望在 shell 里拥有类似 gflags 的处理方式，也就是我们今天要介绍的 [shflags](https://github.com/kward/shflags)

<!--more-->

## 1. 简介

如果用一句话描述清楚 shflags 的话，那就是

> shFlags is a port of the Google gflags library for Unix shell.

使用上跟 gflags 极像，看个例子:

```
#!/bin/sh
#
# This is the proverbial 'Hello, world!' script to demonstrate the most basic
# functionality of shFlags.
#
# This script demonstrates accepts a single command-line flag of '-n' (or
# '--name'). If a name is given, it is output, otherwise the default of 'world'
# is output.

# Source shflags.
. ../shflags

# Define a 'name' command-line string flag.
DEFINE_string 'name' 'world' 'name to say hello to' 'n'

# Parse the command-line.
FLAGS "$@" || exit 1
eval set -- "${FLAGS_ARGV}"

echo "Hello, ${FLAGS_name}!"
```

输出为

```
$ sh hello_world.sh
Hello, world!
$ sh hello_world.sh  --name ying
Hello, ying!
```

解释一下`hello_world.sh`这个脚本

`. ../shflags`导入 shflags 脚本.

`DEFINE_string 'name' 'world' 'name to say hello to' 'n'`定义了一个 flags 变量，其中：

1. 变量名: `name`，使用 `--name` 可以指定变量值`FLAGS_name`.
2. 变量默认值: `world`，即没有指定时的默认值
3. 变量描述: `name to say hello to`
4. short变量名: `n`，使用 `-n` 可以指定变量值`FLAGS_name`

`FLAGS "$@" || exit $?` 这句类似于 `google::ParseCommandLineFlags`，`FLAGS`是内置函数

```
# Parse the flags.
#
# Args:
#   unnamed: list: command-line flags to parse
# Returns:
#   integer: success of operation, or error
FLAGS() {
    ...
}
```

`eval set -- "${FLAGS_ARGV}"` 重新设置了`$@`，留下未解析的参数 `${FLAGS_ARGV}`.

`echo "Hello, ${FLAGS_name}!"` 输出`${FLAGS_name}`.

## 2. 类型

shflags 支持多种类型，当然，底层都是 string.

```
DEFINE_string 'name' 'world' 'name to say hello to' 'n'
DEFINE_boolean 'force' false 'force overwriting' 'f'
DEFINE_integer 'limit' 10 'number of items retuned' 'l'
DEFINE_float 'time' '10.5' 'number of seconds to run' 't'
```

`boolen`类型提前预定义了`FLAGS_TRUE/FLAGS_FALSE`用于比较。

```
# 0
echo ${FLAGS_TRUE}
# 1
echo ${FLAGS_FALSE}
```

注意判断时都使用`-eq -ne -le -lt -ge -gt`，例如

```
if [ ${FLAGS_force} -eq ${FLAGS_FALSE} ] ; then

[ ${FLAGS_debug} -eq ${FLAGS_TRUE} ] || return
```

## 3. FLAGS_HELP

使用 shflags 后，可以自动生成 help 文档，例如

```
$ sh hello_world.sh  --help
USAGE: hello_world.sh [flags] args
flags:
  -n,--name:  name to say hello to (default: 'world')
  -h,--help:  show this help (default: false)
```

如果想要自定义 help，可以重新定义`FLAGS_HELP`

```
FLAGS_HELP=`cat <<EOF
commands:
  speak:  say something
  sing:   sing something
EOF`
```

此外还有很多内置变量，可以用于获取 flags 信息

```
# Shared attributes:
#   flags_error:  last error message
#   flags_output: last function output (rarely valid)
#   flags_return: last return value
#
#   __flags_longNames: list of long names for all flags
#   __flags_shortNames: list of short names for all flags
#   __flags_boolNames: list of boolean flag names
#
#   __flags_opts: options parsed by getopt
#
# Per-flag attributes:
#   FLAGS_<flag_name>: contains value of flag named 'flag_name'
#   __flags_<flag_name>_default: the default flag value
#   __flags_<flag_name>_help: the flag help string
#   __flags_<flag_name>_short: the flag short name
#   __flags_<flag_name>_type: the flag type
```

## 4. Notes

熟悉 gflags 后， shflags 基本是属于上手即用的，介绍就到这了。建议了解下实现，包括函数如何组织、变量命名方式，甚至单测也很完备🐮，不懂的参考下之前的[bash笔记](https://izualzhy.cn/advanced-bash-scripting-guide-booknote)。

尽管这样统一了命令行参数的处理方式，在我看来还是不够完美，因为还是要在不同的地方( shell/c++ )分别定义同名 flags，或许可以有一种通用的配置，类似于 proto 定义的方式，不同语言生成不同的 lib 文件，或者 shflags 能够支持 gflags 里的 --fromenv，都是不错的解决方法。

## 5. References

[wiki of shflags](https://github.com/kward/shflags/wiki)
