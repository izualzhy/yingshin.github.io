---
title:  "高级Bash脚本编程指南读书笔记"
date: 2015-04-18 17:33:59
excerpt: "高级Bash脚本编程指南读书笔记"
tags: bash
---
本文是[《高级Bash脚本编程指南》](http://teliute.org/linux/abs-3.9.1/index.html)的读书笔记。主要记录了自己觉得容易忘记的地方。

<!--more-->

#### 特殊字符
+ 解释器  
只会删除文件自身，不输出`test`

```
#/bin/rm
echo "test"
```
+ 进制

```
# #前的数字即为进制
echo $((2#101011)) #43
echo $((3#101011)) #274
```
+ 变量中的空白

```
# 引用一个变量将保留其中的空白, 当然, 如果是变量替换就不会保留了.
a="A     B C                       D"
#A B C D
echo $a
#A     B C                       D
echo "$a"
```

#### 变量赋值

```
#有两种机制可以进行变量赋值
a=`ls`
a=$(ls)
```

#### bash变量是不区分类型的

Bash并不对变量区分"类型". 本质上, Bash变量都是字符串. 但是依赖于具体的上下文,Bash也允许比较操作和整数操作. 其中的关键因素就是, 变量中的值是否只有数字.

```
a=2334
let a+=1
echo $a #2335
b=${a/23/BB}
echo $b #BB35
let b+=1
echo $b #1
```

#### 特殊的变量类型

{}标记法提供了一种提取从命令行传递到脚本的最后一个位置参数的简单办法. 但是这种方法同时还需要使用间接引用.

```
args=$# #参数的个数
lastarg=${!args} #最后一个参数
```

#### 引用变量

这段关于IFS的代码没有看懂

```
var="'(]\\{}\$\'"
echo $var #(]\{}$\
IFS='\'
echo $var #(] {}$ 
```

#### 转义

注意\转义的解释

```
echo \z #z
echo \\z #\z
echo '\z' #\z
echo '\\z' #\\z
echo "\z" #\z
echo "\\z" #\z
variable=\\z
echo "variable = $variable" #variable = \z
variable="\\z"
echo "variable = $variable" #variable = \z

echo `echo \z` #z
echo `echo \\z` #z
echo `echo \\\z` #\z
echo `echo \\\\z` #\z
echo `echo \\\\\z` #\z
echo `echo \\\\\\z` #\z
echo `echo \\\\\\\z` #\\z
echo `echo "\z"` #\z
echo `echo "\\z"` #\z

#\z
cat << EOF
\z
EOF

#\z
cat << EOF
\\z
EOF

#foo
#bar
echo "foo
bar"
#foobar
echo "foo\
bar"

#只换行：
echo -e '\n'
echo $'\n'
```

#### 条件测试结构

+ 0 is True

```
if [ 0 ]
then
    echo "0 is true."
else
    echo "0 is false."
fi
```
+ `if [ 0 ] if [ 1 ] if [ -1 ] if [ xyz ] if [ "false" ]`都为true，未初始化或者初始化了赋null值则为false(xyz=的赋值形式)
+ `test,/usr/bin/test, [], /usr/bin/[`都是等价命令

```
-bash-3.00$ type test
test is a shell builtin
-bash-3.00$ type [
[ is a shell builtin
-bash-3.00$ type [[
[[ is a shell keyword
-bash-3.00$ type ]
-bash: type: ]: not found
-bash-3.00$ type ]]
]] is a shell keyword
```
+ "if COMMAND"结构将会返回COMMAND的退出状态码，例如`if grep -q ...`
+ 使用[[...]]条件判断结构，而不是[...]，能够防止脚本中的许多逻辑错误，比如，&&,||,<,和>操作符能够正常存在于[[]]条件判断结构中，但是如果出现在[]结构中的话，会报错

```
if  [ "$a" = "$b" ]等价于 if[ "$a" == "$b" ]
[ $a == z* ]      # 文件扩展匹配(file globbing)和单词分割有效，没有找到合适的例子，不理解
[ $a == "z*" ] [[ $a == "z*" ]] 都是取字面意思，即a为"z*"
[[ $a == z* ]] 模式匹配，即a=zufo 返回True 
[[ $a != z* ]]模式匹配，即a=za 返回False

>,<在[]须转义
[[ "$a" > "$b" ]]
[ "$a" \> "$b" ]

#总结：尽量使用[[ ]]
```
+ 检查字符串是否为NULL

```
# str未定义
[ -n $str ] True #注意!
[ -n "$str" ] False
[ -z $str ] True
[ -z "$str" ] True
[ $str ] False
str=abc
[ $str ] True

#在一个混合测试中, 即使使用引用的字符串变量也可能还不够. 如果$string为空的话, [ -n "$string" -o "$a" = "$b" ]可能会在某些版本的Bash中产生错误. 安全的做法是附加一个额外的字符给可能的空变量, [ "x$string" != x -o "x$a" = "x$b" ] ("x"字符是可以相互抵消的).

```

#### 操作符

逻辑操作符  
`if [ $condition1 ] && [ $condition2 ]`  
`if [[ $condition1 && $condition2 ]]`  
`if [ $condition1 -a $condition2 ]`  
注意1,2 &&含义不同

#### 数字常量

shell 默认情况下都是把数字当做10进制处理。除非这个数字采用了特殊的标记或者前缀。  
0表示八进制，0x表示16进制，或者是Base#Number的形式。   
Base的取值范围是2-64,对应的数字表示为10个数字 + 26个小写字母 + 26个大写字母 + @ + \_。   
如果Number超过了Base指定的范围，会报错。   

```
let "dec = 32"
#32
echo "decimal number = $dec"

let "oct = 032"
#26
echo "octal number = $oct"

let "hex = 0x32"
#50
echo "hexadecimal number = $hex"

let "bin = 2#11111111"
#255
echo "binary number = $bin"

let "b32 = 32#32"
#98
echo "base-32 number = $b32"

let "b64 = 64#@_"
#4021=64 * 62 + 63
echo "base-64 number = $b64"

echo

#1295
echo $((36#zz))
#170
echo $((2#10101010))
#44822
echo $((16#AF16))
#3375
echo $((53#1aA))

#./numbers.sh: line 38: let: bad_oct = 081: value too great for base (error token is "081")
let "bad_oct = 081"


```

#### 内部变量

+ 该函数可以打印调用的参数

```
output_args_one_per_line()
{
    for arg
    do echo "[$arg]"
    done
}
IFS=":"
var=":a::b:c::::"
output_args_one_per_line $var
var="ab::cd"
output_args_one_per_line $var
# 自定义的分隔符，如果有多个的话，第一个会被忽略掉？

```
+ `$@ "$@" $*`把每个参数看做单独的单词
+ `"$*"`将所有参数看做一个单词
+ 依赖IFS的设置，`$* $@`会表现出不一致的行为，被例子绕晕了
+ `$_` 上个命令最后个参数
+ `!*` 上个命令的所有参数


#### 操作字符串

+ 字符串求长度

```
str=abcABC123ufoxyz     
#15
echo ${#str}            
#15
echo `expr length $str` 
#15
echo `expr "$str" : '.*'`
#注意最后一个:两边的space
#最后一个其实是：
#匹配字符串开头的子串长度的一种特殊形式，即子串为串本身。
str=abcABC123ABCabc
#8
echo `expr "$str" : 'abc[A-Z]*.2' `
```
+ 提取子串

```
str=abcdefg
echo ${str:3}#defg ${string:position} 返回string 从position到末尾的子串
echo ${str:3:2}#de ${string:position:length} 返回string 从postion开始，长度为length的子串
#如果$string参数是"*"或"@", 那么将会从$position位置开始提取$length个位置参数, 但是由于可能没有$length个位置参数了, 那么就有几个位置参数就提取几个位置参数.
${1:0:1}
echo ${str:-4}#abcdefg 传入负数时，返回串本身
echo ${str: -4}#defg 加入空格或者（）按负索引生成子串
echo ${str:(-4)}#defg

echo `expr substr $str 3 2`#cd
#普通形式为：echo `expr substr $string $position $length`
#注意结果和上面的不同

#按照正则提取子串
expr "$string" : '\(regex\)'
expr match "$string" '\(regex\)'
其中\(...\) 里的内容是要提取的内容
```
+ 子串消除：

```
${string#regex}
${string##regex}
${string%regex}
${string%%regex}
# #表示从前往后匹配regex的被消除
# %表示从后往前匹配regex的被消除
# ##,%%表示贪心，匹配越多越好，否则匹配到就返回。
```
+ 子串替换：

```
# ${string/substring/replacement}替换string第一个匹配replacement的子串为substring
# ${string//substring/replacement}替换string所有匹配replacement的子串为substring
# ${string/#substring/replacement}如果substring匹配string的开头部分，就替换为replacement
# ${string/%substring/replacement}如果substring匹配string的结尾部分，就替换为replacement.

str="abcufoABCabc123"
# ABCufoABCabc123
echo ${str/abc/ABC}
# ABCufoABCABC123
echo ${str//abc/ABC}
# abcufoABCabcABC
echo ${str/%123/ABC}
# ABCufoABCabc123
echo ${str/#abc/ABC}
```

#### 参数替换

```
${parameter}与$parameter相同, 也就是变量parameter的值. 在某些上下文中,${parameter}很少会产生混淆.

${parameter-default}, ${parameter:-default}
${parameter-default} -- 如果变量parameter没被声明, 那么就使用默认值.
${parameter:-default} -- 如果变量parameter没被设置, 那么就使用默认值.

${parameter=default}, ${parameter:=default}
${parameter=default} -- 如果变量parameter没声明, 那么就把它的值设为default.
${parameter:=default} -- 如果变量parameter没设置, 那么就把它的值设为default.

${parameter+alt_value}, ${parameter:+alt_value}
${parameter+alt_value} -- 如果变量parameter被声明了, 那么就使用alt_value, 否则就使用null字符串.
${parameter:+alt_value} -- 如果变量parameter被设置了, 那么就使用alt_value, 否则就使用null字符串.

${parameter?err_msg}, ${parameter:?err_msg}
${parameter?err_msg} -- 如果parameter已经被声明, 那么就使用设置的值, 否则打印err_msg错误消息.
${parameter:?err_msg} -- 如果parameter已经被设置, 那么就使用设置的值, 否则打印err_msg错误消息.

不加：与声明相关，加：与设置相关，因为没有设置包含没有声明这个条件，所以加了：影响范围更大。
只有在parameter被声明同时设置为null时，：会有不同。

${!varprefix\*}, ${!varprefix@}
匹配所有之前声明过的, 并且以varprefix开头的变量.
```

#### 变量的间接引用

`\$$var `或者`${!var}`的形式

```
y=x
x=abc
eval y=\$$y
#abc
echo $y
#abc
echo ${!y}
```

#### $RANDOM: 产生随机整数

RANDOM是BASH的内建函数，返回[0, 32767]间的伪随机数。虽然是函数，但是也可以这么用:

```
echo $((RANDOM+1))
993
```
awk也可以产生一个[0, 1]的随机数

```
echo | awk '{srand();print rand()}'
0.744095
```

#### 循环

```
for arg in [list]
do
     commands
done
# 如果在同一行，要加个分号
for arg in [list]; do
如果没有in [list]，那么循环将操作$@，即从命令行传给脚本的位置参数。
```
(())的循环结构

```
LIMIT=10
for ((a=1; a<=LIMIT; a++))#LIMIT没有使用$
do
    echo $a
done
```

#### 循环控制

break命令可以带一个参数，一个不带参数的break命令只能退出当前所在（最内层）的循环，而break N可以跳出N层循环 __continue N没有理解__

```
for outerloop in 1 2 3 4 5
do
    echo "Group $outerloop: "

    for innerloop in 1 2 3 4 5
    do
        echo -n "$innerloop "
        if [ "$innerloop" -eq 3 ]
        then
            break 2
        fi
    done
done
#Group 1: 
#1 2 3
```

#### 基本命令

+ ls -S按照文件尺寸列出文件
+ -exec COMMAND \;
find ~ -name "core\*" -exec rm -rf {} \;
1. find 用所有匹配文件的路径名来替换{}
2. 命令需要以;结束
+ 循环输出的话，watch比sleep更加简洁。比如每隔60s查看一次log
watch -n 60 'grep "\[E" server\*.log | tail'

#### 文本处理命令

+ 先排序，再统计，再按照出现的次数DESC排列。这个原型常用于统计log或者词典列表。`sort INPUTFILE | uniq -c | sort -nr`
+ column -t可以转化为易于打印的表的形式，例如：

```
-bash-3.00$ ./test.sh
123 asdf 9.9
123456 ufo 1.0
1 playernumber 1.11111
-bash-3.00$ ./test.sh | column -t
123     asdf          9.9
123456  ufo           1.0
1       playernumber  1.11111
```
+ cat -n会显示行号，nl的输出类似，但是空行前不显示行号

#### 文件与归档命令

+ strings可以从二进制或者数据文件中读到可打印字符
+ diff --side-by-side会按照左右分隔的形式比较文件  
-r 递归   
-N 确保补丁文件将正确的处理已经创建或删除文件的情况   
-u 输出每个修改的前后3行。也可以用-u5指定输出更多上下文   
+ patch -pNum <patchfile>   
-E 如果发现了空文件，那么就删除它  
-R 取消打过的补丁  
-p Num忽略几层文件夹   
比如如下的patch文件片段  

```
--- old/modules/pcitable       Mon Sep 27 11:03:56 1999  
+++ new/modules/pcitable       Tue Dec 19 20:05:41 2000  
```
如果使用参数 -p0，那就表示从当前目录找一个叫做old的文件夹，再在它下面寻找 modules/pcitable 文件来执行patch操作。  
而如果使用参数 -p1，那就表示忽略第一层目录（即不管old），从当前目录寻找 modules 的文件夹，再在它下面找pcitable。  

+ diff/patch应用：

```
单个文件：  
diff -uN from-file to-file > to-file.patch #产生补  丁
patch -p0 < to-file.patch #打补丁  
patch -RE -p0 < to-file.patch #取消补丁  
文件夹：  
diff -uNr from-dir to-dir > to-dir.patch > #产生补丁  
cd to-dir > #打补丁  
patch -p1 < to-dir.patch  
patch -R -p1 < to-dir.patch > #取消补丁  
```
+ cmp是diff的一个简单版本，相同则返回0，不同则返回1。
+ shred 安全的删除一个文件，使用随机字符填充目标文件
+ tempfile=`mktemp $PREFIX.XXXXXX`
创建一个随机的文件名($PREFIX固定 XXXXXX替换为随机的6个字符），完整的文件名储存在tempfile.

#### 数学计算命令

+ factor分解素数
+ Bash不能处理浮点运算
    1. 需要使用bc
    `variable=$(echo "OPTIONS; OPERATIONS" | bc)`
    例如`var=$(echo "scale=9; 1.0+2.2" | bc)`
    2. here document的形式

    ```
    -bash-3.00$ echo `bc << EOF
    > 1.1 + 2.2
    > EOF
    > `
    3.3
    -bash-3.00$ echo $(bc << EOF
    > 1.1 + 2.2
    > EOF
    > )
    3.3
    ```

#### 混杂命令

seq产生出来的整数一般都占一行，-s可以修改为空格、冒号。

#### 系统与管理命令

nmap(Network Mapper) 可以端口扫描  
nc(netcat)可以连接和监听TCP/UDP端口

#### 命令替换

```
variable=`< file`
variable=`cat file`
```
+ 以上都会读取file的内容到variable，但是第二行会fork一个新进程，所以比第二行执行慢很多
+ $( )与\`\`的不同：

```
#1. $()允许嵌套
var=`echo `echo adsf``
#adsf: command not found
var=$(echo $(echo asf))
echo $var
#asf
2. 处理双反斜线时不同
-bash-3.00$ echo `echo \\`

-bash-3.00$ echo $(echo \\)
\
```

#### 算术扩展

```
echo `expr $z + 3`#注意+两边的空格
echo $(($z+3))
echo $((z+3))#使用双括号的形式，参数解引用是可选的
```

#### I/O重定向

n<&- 关闭输入文件描述符n  
n>&- 关闭输出文件描述符n  
i>&j 重定向文件描述符i到j，指向i文件的所有输出都发送到j.  

####使用exec

`exec 6<&0` 将文件描述符6与stdin链接起来，保存stdin.  
`exec < data-file` stdin被文件'data-file'代替  
`exec 0<&6 6<&-` #将stdin从fd #6恢复，并且关闭fd #6.  
写到0的输出都会写到6，因此恢复时只要把写到6的输出都写到0就可以了。  

`exec 6>&1`将文件描述符6与stdout链接起来，保存stdout  
`exec > data-file`stdout被文件'data-file'代替  
`exec 1>&6 6>&-` 将stdout从fd #6恢复，并且关闭fd #6.  
写到6的输出都会写到1，因此恢复时只要把写到1的输出都写到6就可以了。  

我理解的0, 1, 2都分别始终与键盘，屏幕，屏幕绑定  

#### 通配(globbing)

bash使用的并不是标准的RE，仅仅使用通配符。

#### 子shell
(command1;command2;command3...)  
圆括号里的命令将会在子shell中运行  
子shell里的变量在父shell里无效，目录的更改也不会影响父shell  

#### 复杂函数和函数复杂性

函数所能返回的最大值为255

```
return_test ()
{
    return $1
}

return_test 2
#2
echo $?
return_test 255
#255
echo $?
return_test 256
#0
echo $?
#如果想返回更大的值，可以echo然后命令替换捕捉该值
```

#### 数组

+ 数组的赋值方法有三种:
    1. `array[0]=xxx`
    2. `array=(xxx xxx xxx)`
    3. `array=([0]=xxx [2]=xxx [100]=xxx)`
+ bash允许把变量当成数组来操作，即使这个变量没有被声明为数组。数组中只有一个元素，就是这个变量本身.

```
${array[i]} 取元素
${#array[@]}  ${#array[*]} 取数组元素个数
${array[@]} ${array[*]} 取整个数组
```
大部分标准字符串操作可以用于数组中，不过在数组替换与截断时是对每个元素进行操作：`array=(zero one two three four five five)`
+ 数组元素偏移

```
echo ${array[@]:2} #two three four five five
echo ${array[*]:1} #one two three four five five
echo ${array[@]:2:2} #two three
```
+ 数组元素截断

```
echo ${array[@]#f*r} #zero one two three five five
echo ${array[@]#*four} #zero one two three five five
echo ${array[@]%th*e} #zero one two four five five
```
+ 数组元素替换

```
#对每个元素操作，所以以下两行结果一样
echo ${array[@]/fiv/XYZ} #zero one two three four XYZe XYZe
echo ${array[@]//fiv/XYZ} #zero one two three four XYZe XYZe
echo ${array[@]//fi/} #zero one two three four ve ve
echo ${array[@]/%ve/ZZ} #zero one two three four fiZZ fiZZ
echo ${array[@]/#fi/XY} #zero one two three four XYve XYve
echo ${array[@]/%o/XX} #zerXX one twXX three four five five
```

#### 调试

+ `sh -n scriptname` 不会运行脚本，只会检查脚本的语法错误。
+ 捕获信号： trap
trap 'echo "Control-C disabled."' 2
按下Control-C后，只会打印上述语句.
