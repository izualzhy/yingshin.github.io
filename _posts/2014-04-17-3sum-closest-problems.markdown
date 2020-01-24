---
title:  "leetcode上3Sum Closest问题的分析"
date: 2014-04-17 18:21:52
excerpt: "3Sum Closest problems on leetcode."
tags: sort  leetcode
---

与[这篇文章](http://izualzhy.cn/ksum-problems/)的思路非常类似，题目链接如下：  
[3Sum Closest ](http://oj.leetcode.com/problems/3sum-closest/)  
**Given an array S of n integers, find three integers in S such that the sum is closest to a given number, target. Return the sum of the three integers. You may assume that**


还是排序之后的操作，3Sum Closest的问题可以转化为2Sum Closest的问题。 
先考虑下两个数的情况，使得和与target最接近。并返回这个和。  
计算思路如下：  
数组排序，首尾两个指针。  
每次和都被用于比较更新最接近target的值。  
如果比target大，那么尾指针-1,有可能会更接近target。 
如果比target小，那么首指针-1,有可能会更接近target。
如果相等，则直接返回。  

代码如下：  

```
int twoSumClosest(vector<int> &num, int target, int start)
{
    int left = start, right = num.size() - 1;
    int minSum = 0xffffff, pre = 0xffffff;
    while (left < right)
    {
        int sum = num[left] + num[right];

        if (abs(sum - target) < abs(minSum - target))
            minSum = sum;

        pre = sum;

        if (sum > target)
            --right;
        else if (sum < target)
            ++left;
        else
            return minSum;
    }

    return minSum;
}

int threeSumClosest(vector<int> &num, int target)
{
    sort(num.begin(), num.end());
    int minSum = 0xffffff;

    for (int i = 0; i < num.size() - 2; ++i)
    {
        int sum = twoSumClosest(num, target - num[i], i + 1);
        if (abs(minSum - target) > abs(sum + num[i] - target))
            minSum = num[i] + sum;
    }

    return minSum;
}
```
