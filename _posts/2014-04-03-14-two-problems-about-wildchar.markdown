---
title:  "两个有关通配符的问题"
date:   2014-03-14 17:31:03
excerpt: "一个问题来自leetcode,用于比较一个通配符字符串能否与另一个普通字符串匹配，另一个问题来自英雄会，计算符合通配符字符串比一个普通字符串大的字符串的总个数"
tags: algorithm
---
看到两个比较有意思的题目，都跟通配符相关。
地址在这里   

1. [Implement wildcard pattern matching with support for '?' and '\*'.][1] 
2. [有多少个整数符合W的形式并且比X大？][2]  

**其中[1][1]个人感觉更复杂一些。**


[1][1]的难度主要在\*号,?可以直接匹配一个目标字符串并跳过，\*考虑其后地一个非\*字符在目标字符串里的匹配位置。  
[2][2]用递归就可以了，要想一个数字比另一个更大，，从高位开始选取，高位大则低位可以任意取值，相等则低位需要大于或者相等，小于那么不符合。前两种情况
可以递归，从高位到下一位的过程。

<!--more-->

[1][1]代码如下:

```
//http://oj.leetcode.com/problems/wildcard-matching/
#include <stdio.h>
#include <string.h>

bool isMatch(const char* s, const char* p)
{
    int slen = strlen(s);
    int plen = strlen(p);
    //k记录ｓ里匹配ｐ里*后的第一个非*的索引.
    int i = 0, j = 0, k = -1;
    int lastStarIndexInS = -1;
    int nextCharIndexInP = -1;
    while (i < slen)
    {
        //pattern reach end.backward if * exists.
        if (j >= plen)
        {
            if (lastStarIndexInS >= 0)
            {
                j = lastStarIndexInS;
                i = k + 1;
            }
        }

        if (p[j] == '*')
        {
            //记录*位置
            lastStarIndexInS = j;
            //查找*后地一个非*的位置，记录在nextCharIndexInP.
            if (nextCharIndexInP < j)
            {
                nextCharIndexInP = j;
                while (nextCharIndexInP < plen && p[nextCharIndexInP] == '*')
                    ++nextCharIndexInP;
                if (nextCharIndexInP == plen)
                    return  true;
            }
            else
            {
                //如果已经找到了nextCharIndexInP,不断增加i直到查找到ｓ里第一个匹配的位置，记录在ｋ
                if (s[i] == p[nextCharIndexInP] || p[nextCharIndexInP] == '?')
                {
                    k = i;
                    j = nextCharIndexInP + 1;
                }
                ++i;
            }
        }
        //字符相等或者?，都认为相等，继续判断。
        else if (s[i] == p[j] || p[j] == '?')
        {
            ++i;
            ++j;
        }
        else
        {
            //不相等则回退到上个*的位置（如果存在的话），同时i从ｋ，即匹配的位置查找与p[nextCharIndexInP]相等的。
            if (lastStarIndexInS >= 0)
            {
                j = lastStarIndexInS;
                i = k + 1;
            }
            //如果不存在*说明不匹配
            else
            {
                return false;
            }
        }
    }

    //ｓ已经匹配完毕，而ｐ还有多个*,认为相等。
    while (j < plen && p[j] == '*')
    {
        ++j;
    }
    return i == slen && j >= plen;
}

int main()
{
    printf("%s false\n", isMatch("a", "aa") ? "true" : "false");
    printf("%s true\n", isMatch("c", "*?*") ? "true" : "false");
    printf("%s true\n", isMatch("hi", "*?") ? "true" : "false");
    printf("%s true\n", isMatch("acabc", "*c") ? "true" : "false");
    printf("%s true\n", isMatch("geeks", "g*ks") ? "true" : "false");
    printf("%s false\n", isMatch("geeksforgeeks", "ge?ks") ? "true" : "false");
    printf("%s false\n", isMatch("pqrst", "*pqrs") ? "true" : "false");
    printf("%s true\n", isMatch("abcdhghgbcd", "abc*bcd") ? "true" : "false");
    printf("%s false\n", isMatch("abcd", "abc*c?d") ? "true" : "false");
    printf("%s true\n", isMatch("abcd", "*c*d") ? "true" : "false");
    printf("%s false\n", isMatch("abcd", "?c*d") ? "true" : "false");
    printf("%s true\n", isMatch("abcd", "*?c*d") ? "true" : "false");

    return 0;
}

```

[2][2]代码如下：

```
//http://hero.csdn.net/Question/Details?ID=351&ExamID=346
#include <stdio.h>
#include <string.h>

/*给定一个带通配符问号的数W，问号可以代表任意一个一位数字。*/
/*再给定一个整数X，和W具有同样的长度。*/
/*问有多少个整数符合W的形式并且比X大？*/

/*输入格式*/
/*多组数据，每组数据两行，第一行是W，第二行是X，它们长度相同。在[1..10]之间.*/
/*输出格式*/
/*每行一个整数表示结果。*/
char W[100], X[100];

/**
 * @brief count 计算多少个整数符合w形式且比ｘ大
 *
 * @param w 带通配符问好的数
 * @param x 整数
 * @param any 1表示遇到？可以选择任意数，0表示必须比ｘ上对应位置的数字大
 *
 * @return  返回符合的整数个数
 */
long long int count(const char* w, const char* x, long long int any)
{
    long long int len = strlen(w);
    long long int res = len || any ? 1 : 0;//空的字符串，任意选取为1,否则为0.
    if (len)
    {
        if (w[0] == '?')
        {
            if (any)
            {
                //可选择任意数，［0-9］共10个数字
                res = 10 * count(w + 1, x + 1, 1);
            }
            else
            {
                //不可选择，分>,=两种情况，>则接下来可以任意选取，=需要选择。
                res = count(w + 1, x + 1, 1) * ('9' - x[0]) + count(w + 1, x + 1, 0);
            }
        }
        //不是？则取决于接下来的比较
        else if (any)
        {
            //不是？且随意选取则对结果没有影响
            res = count(w + 1, x + 1, 1);
        }
        else
        {
            //接下来可以随意选取
            if (w[0] > x[0])
            {
                res = count(w + 1, x + 1, 1);
            }
            //接下来需要选择，因为高位没有>
            else if (w[0] == x[0])
            {
                res = count(w + 1, x + 1, 0);
            }
            //高位已经<，返回数量0个。
            else
            {
                res = 0;
            }
        }
    }

    return res;
}

int main()
{
    while(~scanf("%s\n%s", W, X))
    {
        printf("%lld\n", count(W, X, 0));
        memset(W, 0, sizeof(W));
        memset(X, 0, sizeof(X));
    }

    return 0;
}
```



[1]: http://oj.leetcode.com/problems/wildcard-matching/
[2]: http://hero.csdn.net/Question/Details?ID=351&ExamID=346
