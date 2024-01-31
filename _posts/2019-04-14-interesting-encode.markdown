---
title: "神奇的锟斤拷"
date: 2019-04-14 00:03:08
tags: mixed
---

上个周末接到电话，说是线上有“锟斤拷”的结果出现，不再做电子书解析这些事情后，对这类问题逐渐淡忘了，这篇笔记就介绍下神奇的“锟斤拷”是如何产生的。

很多系统间的转换，例如编码的转换、时间的转换等，在我看来也是程序员的基本技能之一，比如当你开发一个评论系统时，不同时区的人回复后，应当怎么样记录这些时间，并且提供给别人用的时候最大可能的保证不会出错？

关于编码，有一次让我印象很深刻的 code review，之前在多看的时候，大概写了这么一段代码和注释：

```
//最多允许填充63个字符或者31个汉字
char buffer[64];
...
```

场景模糊记得是直接设置返回到前端一些提示语，用于在线更新时的提示，所以额外注释了下能写多少个汉字。code review 给的意见是希望限制的字数应当统一，无论对于字符还是汉字，这样设置更新提示语时更友好些。于是我花了几个小时看了下编码相关的知识，不再是想当然的认知。

## 1. U+FFED

在一个 gbk 的终端，试着执行下这段 python 程序:

```python
>>> print u'\uFFFD'.encode('utf-8')*2
锟斤拷
```

结果就这么产出了😅

接下来，介绍下我的理解。

## 2. ASCII -> Unicode

`man ascii`可以看到 ASCII 编码:

```
NAME
       ascii - the ASCII character set encoded in octal, decimal, and hexadecimal

DESCRIPTION
       ASCII  is  the  American  Standard  Code for Information Interchange.  It is a 7-bit code.  Many
       8-bit codes (such as ISO 8859-1, the Linux default character set) contain ASCII as  their  lower
       half.  The international counterpart of ASCII is known as ISO 646.

       The following table contains the 128 ASCII characters.

       C program '\X' escapes are noted.

       Oct   Dec   Hex   Char                        Oct   Dec   Hex   Char
       ------------------------------------------------------------------------
       000   0     00    NUL '\0'                    100   64    40    @
       001   1     01    SOH (start of heading)      101   65    41    A
       002   2     02    STX (start of text)         102   66    42    B
       003   3     03    ETX (end of text)           103   67    43    C
       004   4     04    EOT (end of transmission)   104   68    44    D
       005   5     05    ENQ (enquiry)               105   69    45    E
       006   6     06    ACK (acknowledge)           106   70    46    F
       007   7     07    BEL '\a' (bell)             107   71    47    G
       010   8     08    BS  '\b' (backspace)        110   72    48    H
       011   9     09    HT  '\t' (horizontal tab)   111   73    49    I
       012   10    0A    LF  '\n' (new line)         112   74    4A    J
       013   11    0B    VT  '\v' (vertical tab)     113   75    4B    K
       014   12    0C    FF  '\f' (form feed)        114   76    4C    L
       015   13    0D    CR  '\r' (carriage ret)     115   77    4D    M
       016   14    0E    SO  (shift out)             116   78    4E    N
```

一共128个字符，每个字符占一个字节，包含了大小写字符和常见的符号，例如制表符 空格 回车 换行等，最大为 0X7F(01111111).

作为 American Standard Code for Information Interchange, ASCII 最开始是足够用的，可是随着字符的增多，128个字符不够用了，于是扩充到了256。

此时仍然可以只用一个字节表示(最大0XFF)。

但是一个字节能够表示的字符是有限的，对于汉字来讲肯定不够，于是我们创造了[GBK编码](https://zh.wikipedia.org/wiki/%E6%B1%89%E5%AD%97%E5%86%85%E7%A0%81%E6%89%A9%E5%B1%95%E8%A7%84%E8%8C%83)，规定了每两个字节表示一个汉字，具体的编码表可以参考[这里](http://ff.163.com/newflyff/gbk-list/)，锟斤拷对应的编码分别为：

```
锟 -> EFBF
斤 -> BDEF
拷 -> BFBD
```

文字转换为编码，是计算机处理文字的第一步条件，简单讲只是需要一张“文字” <-> “编码”的映射表。因此很快的，如同雨后春笋般，各种编码格式开始出现，例如处理中文的，有 gb2312 gbk gb18030 big5 等，处理日文的 ecu-jp 等。这些编码格式很快解决了本国文字处理的问题，但是由于没有考虑到兼容性，也同样暴露出来两个问题：

1. 同样两个字节，在两种语言里都有用来表示一个字，那么以哪个为准？  
2. 不同编码之间怎么转换？  

在这样的背景下，Unicode 产生了.

## 3. Unicode

[Unicode](https://zh.wikipedia.org/wiki/Unicode)是一种编码的标准，其目的是希望通过一套编码来包含地球上所有字符，严格规定了哪种语言的文字占用哪些字节范围，你可以在[这里](https://unicode-table.com/en/#control-character)查询到一个完整的记录。

这样，我们就可以用一条编码字符集，而不是在多套之间转换。

中文字符可以在[这里](http://www.chi2ko.com/tool/CJK.htm)查询到，例如：

```
百度 = 767E 5EA6
```

## 4. UTF-8

Unicode 只是规定了编码的标准，但是没有说明存储的格式，例如对于上面“百”的编码: 0X767E，同样也可以理解为是两个字符(v~)的编码。

```
>>> print '\x76\x7E'
v~
```

所以要想正确的存储和传输 Unicode，还需要一套 Unicode 转换格式（Unicode Transformation Format，简称为UTF）。

UTF-8 UTF-16 都可以转换，为了避免太发散，我们讲下 UTF-8.

不同于 gbk，UTF-8 是一种变长编码格式，比如对于 acsii 字符，还是使用一个字节表示。对于汉字，则使用三个字节。

具体的转换方式为：

|Unicode/UCS-4  |bit数  |UTF-8  |byte数  |
|--|--|--|--|
|0000~007F  |0~7  |0XXX XXXX  |1  |
|0080~07FF  |8~11  |110X XXXX 10XX XXXX  |2  |
|0800~FFFF  |12~16  |1110 XXXX 10XX XXXX 10XX XXXX  |3  |
|1 0000~1F FFFF  |17~21  |1111 0XXX 10XX XXXX 10XX XXXX 10XX XXXX |4  |
|20 0000~3FF FFFF  |22~26  |1111 10XX 10XX XXXX 10XX XXXX 10XX XXXX 10XX XXXX  |5  |
|400 0000~7FFF FFFF  |27~31  |1111 110X 10XX XXXX 10XX XXXX 10XX XXXX 10XX XXXX 10XX XXXX  |6  |

*注：从低位到高位依次填充，不足补0*

例如对于“百度”(767E 5EA6):

```
767E = 0b111011001111110 -> 111 011001 111110 -> 11100111 10011001 10111110 -> E799BE
5EA6 = 0b101111010100110 -> 101 111010 100110 -> 11100101 10111010 10100110 -> E5BAA6
```

计算出的就是“百度”对应的 UTF-8 编码了：

```
>>> print '\xE7\x99\xBE\xE5\xBA\xA6'
百度
```

*注：终端需要是utf-8环境*

## 5. Specials (Unicode block)

Unicode 有一些[特殊字符](https://en.wikipedia.org/wiki/Specials_(Unicode_block))，例如 U+FFFD，用于解决系统间字符集可能不一致的问题。

这个字符用 UTF-8 表示就是：

```
FFFD = 0b1111111111111101 -> 1111 111111 111101 -> 11101111 10111111 10111101 -> EFBFBD
```

比如两个系统间 UTF-8 字符集不一致，或者处理格式不正确的几个字节时，可能就用这个字符(0xFFFD)表示：�

比如我们正常存储一个 Unicode 字符串时：

```
>>> print unicode('\xE7\x99\xBE\xE5\xBA\xA6', 'utf-8', 'replace')
百度
```

我们故意颠倒下字节的顺序，制造一个错误：

```
>>> print unicode('\xBE\x99\xE7\xA6\xBA\xE5', 'utf-8', 'replace')
��禺�
>>> unicode('\xBE\x99\xE7\xA6\xBA\xE5', 'utf-8', 'replace')
u'\ufffd\ufffd\u79ba\ufffd'
```

我没有用过具体的 web 框架，不过可以从这节猜测下 U+FFFD 的产生场景。

## 6. 结论

Unicode 编码下，如果遇到未知的字符那么可能用一个 REPLACEMENT 字符替换，这个字符就是 0xFFFD，用 UTF-8 编码就是 0XEFBFBD.

如果出现了字节错误问题，可能会产生多次替换，如果两个 0XFFFD 连续出现，对应就是 0XEFBFBDEFBFBD.

如果这些字节一直用 UTF-8 编码解释和传输，效果上也符合预期。但当这些字节原样通过 GBK 编码解释后就出现问题了。GBK 是双字节编码，其中“锟斤拷”对应的编码为 0XEFBF 0XBDEF 0XBFBD.

这就是神奇的“锟斤拷”的由来。

同样的，可以解释“烫烫烫”等问题。正所谓：

手持两把锟斤拷，口中疾呼烫烫烫。  
脚踏千朵屯屯屯，笑看万物锘锘锘。  
