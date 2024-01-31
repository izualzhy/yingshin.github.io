---
title:  "two word break problems"
date:   2014-04-11 18:19:03
excerpt: "Word Break"
tags: algorithm
---

关于DP的两道很有意思的题目

题目链接如下：   
[Word Break](http://oj.leetcode.com/problems/word-break/)  
> 给定一个字符串s和一个单词表dict,检查dict的单词是否可以组成s,成功返回true,否则false.
> 例如,s="leetcode",dict=["leet", "code"],返回true,因为s可以分成"leet code".

[Word Break II](http://oj.leetcode.com/problems/word-break-ii/)  
> 给定一个字符串s和一个单词表dict,检查dict的单词是否可以组成s,返回所有的组合形式.
> 例如,s="catsanddog",dict=["cat", "sand", "dog", "cats", "and"]
> 返回["cats and dog", "cat sand dog"]。

<!--more-->

对题目一，最直接的想法是递归。  
伪代码如下：  

```
bool check(s, left, right):
    if (s.substr(left, right) in dict):
        return true;

    for i in (left, right):
        if (check(s, left ,i) && check(s, i+1, right)):
            return true;

    return false;
```

比如对单词*leetcode*:  
1. 检查*l*, *eetcode*.  
2. 检查*le*, *etcode*.  
3. ......

其中第一步又需要递归检查字符串*l*,*eetcode*。  
检查*eetocde*:  
1. 检查*e*,*etcode*
2. ......

可以看到这一步的**1**跟上面的**2**重复检查了`etcode`。  
因此递归是比较慢的。  
一个直观的想法是记录`etcode`的检查结果，以后直接使用该值。  
这就是动态规划的想法了。  

建立一个动态规划数组`dp[i]`,表示ｓ的子字符串[i,s.size)是否可以由dict的单词组成。  
子问题的递推解决为:  
子字符串[i,s.size)的值，取决于能否在其中找到一个j,使得[i,j)在dict里，而[j,s.size)可以由dict单词组成。  
实际上是一个求解一个矩阵右上角部分的过程，求解方向从下往上。  
同时跟背包问题类似，可以使用一个一维数组来代替二维数组节省空间。   
具体代码如下：  

```
bool wordBreak(string s, unordered_set<string>& dict)
{
    int len = s.size();
    vector<bool> dp(len + 1, false);//dp[i]表示字符串s从i处开始到结尾的子字符串是否可以有dict的单词组成。
    dp[len] = true;

    //有后往前算
    for (int i = len - 1; i >= 0; --i)
        //j用来计算前半段字符串的结尾
        for (int j = i + 1; j <= len; ++j)
        {
            //如果[j,len)可以组成，同时[i,j)在dict中，则[i,len)在dict中。
            if (dp[j] && dict.find(s.substr(i, j - i)) != dict.end())
            {
                dp[i] = true;
                break;
            }
        }

    return dp[0];
}
```

解决了第一个问题，第二个就好理解了，dp数组记录的是由dict分解后的字符串。  
如果找到了结果，不立刻返回。而是继续查看是否还有别的组合形式。  
代码如下：   

```
vector<string> wordBreak(string s, unordered_set<string>& dict)
{
    int len = s.size();
    //dp[i]表示[i,len)在dict可以找到的组合形式
    vector<vector<string> > dp(len + 1, vector<string>());
    for (int i = len - 1; i >= 0; --i)
        //对每一个i,往后寻找可以划分的组合
        for ( int j = i + 1; j <= len; ++j)
        {
            //dict里存在[i,j)
            if (dict.find(s.substr(i, j - i)) != dict.end())
            {
                //[j,len)有组合的形式
                if (!dp[j].empty())
                {
                    for (unsigned int p = 0; p < dp[j].size(); ++p)
                    {
                        string str = s.substr(i,j - i).append(1, ' ').append(dp[j][p]);
                        dp[i].push_back(str);
                    }
                }
                //j==len,即当前查找的是[i,len),直接加入，不用拼接字符串了。
                else if (j == len)
                {
                    dp[i].push_back(s.substr(i, j - i));
                }
            }
        }

    return dp[0];
}
```

