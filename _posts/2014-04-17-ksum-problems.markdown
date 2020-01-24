---
title:  "leetcode上2Sum 3Sum 4Sum以及kSum问题的分析"
date: 2014-04-17 17:52:21
excerpt: "TwoSum,3Sum,4Sum,kSum problems on leetcode."
tags: sort  leetcode
---

leetcode上很多题目是一系列的，循序渐进，比如TwoSum,3Sum,4Sum的问题。  

1. [TwoSum](http://oj.leetcode.com/problems/two-sum/)   
 **Given an array of integers, find two numbers such that they add up to a specific target number.**
2. [3Sum](http://oj.leetcode.com/problems/3sum/)   
 **Given an array S of n integers, are there elements a, b, c in S such that a + b + c = 0? Find all unique triplets in the array which gives the sum of zero.**
3. [4Sum](http://oj.leetcode.com/problems/4sum/)  
 **Given an array S of n integers, are there elements a, b, c, and d in S such that a + b + c + d = target? Find all unique quadruplets in the array which gives the sum of target.**

要求也很简单，找到给定数组的k个数，使得这ｋ个数和为给定的数值target.  

附加的则是一些计算index,求所有满足的组合，去重的操作等。

<!--more-->

#### 2Sum

先从最简单的**TwoSum**开始。  
如果是杂乱无序的一组数，求和不得不要遍历所有的数值。因此第一想到的就是排序。  
观察下排序后的数组  

`A < B < C < D`  

如果A + B = target，那么A+C,B+D都不可能等于target，也就是说只有B + C可能等于target。 

因此可以这么计算：   


```
while (left < right):
    sum = a[left] + a[right]
    if sum < target: #和太小，增大left.
        ++left
    else if sum > target: #和太大，减小right.
        --right
    else #找到一组
        ++left
        --right
```

写成代码如下：  

```
vector<int> twoSum(vector<int> &numbers, int target)
{
    vector<int> vec = numbers;
    sort(numbers.begin(), numbers.end());
    vector<int> ans;
    int left = 0, right = numbers.size() - 1;
    while (true)
    {
        int sum = numbers[left] + numbers[right];
        if (sum > target)
            --right;
        else if (sum < target)
            ++left;
        else
            break;
    }

    //原题要求返回index,此处计算index.
    int p = -1,q = -1;
    for (int i = 0; i < vec.size(); ++i)
    {
        if (vec[i] == numbers[left] || vec[i] == numbers[right])
        {
            if (p < 0)
                p = i;
            else
                q = i;
        }
    }
    ans.push_back(p + 1);
    ans.push_back(q + 1);

    return ans;
}
```

#### 3Sum

解决了TwoSum,3Sum就好理解了，数组排好序后，取定一个值，然后计算该值后面满足sum的两个数即可。  
可以转化为解决TwoSum的问题。  
注意多了一个去重的要求，因为是排好序的数组，去重只要做到临近的数字相同的话略过即可。  
代码如下：  

```
vector<vector<int> > threeSum(vector<int> &num)
{
    vector<vector<int> > ans;
    if (num.size() < 3)
        return ans;
    sort(num.begin(), num.end());

    for (unsigned int i = 0; i < num.size(); ++i)
    {
        if (i != 0 && num[i] == num[i-1])
            continue;

        int target = -num[i];
        int left = i + 1, right = num.size() - 1;
        while (left < right)
        {
            int sum = num[left] + num[right];
            if (sum < target)
            {
                ++left;
            }
            else if (sum > target)
            {
                --right;
            }
            else
            {
                vector<int> v;
                v.push_back(num[i]);
                v.push_back(num[left]);
                v.push_back(num[right]);
                ans.push_back(v);

                ++left;
                while (left < right && num[left] == num[left - 1])
                    ++left;
                --right;
                while (left < right && num[right] == num[right + 1])
                    --right;
            }
        }
    }

    return ans;
}
```

#### 4Sum以及kSum

到这里其实发现kSum不过是一个递归的问题,要求k个数之和为target[k],可以转化为k-1个数之和target[k-1]。  
代码如下:  

```
//make sure num is sorted in non-descending order before this function.
vector<vector<int> > kSum(vector<int> &num, int left, int target, int k)
{
    vector<vector<int> > ans;
    if (num.size() < k)
        return ans;
    if (k == 2)
    {
        int i = left, j = num.size() - 1;
        while (i < j)
        {
            int sum = num[i] + num[j];
            if (sum < target)
            {
                ++i;
            }
            else if (sum > target)
            {
                --j;
            }
            else 
            {
                vector<int> v;
                v.push_back(num[i]);
                v.push_back(num[j]);
                ans.push_back(v);

                ++i;
                while (i < j && num[i] == num[i-1])
                    ++i;
                --j;
                while (i < j && num[j] == num[j + 1])
                    --j;
            }
        }

        return ans;
    }

    for (unsigned int i = left; i <= num.size() - k; ++i)
    {
        if (i != left && num[i] == num[i - 1])
            continue;

        vector<vector<int> > vec = kSum(num, i + 1, target - num[i], k - 1);
        for (unsigned int j = 0; j < vec.size(); ++j)
        {
            vector<int> v = vec[j];
            v.insert(v.begin(), num[i]);
            ans.push_back(v);
        }
    }

    return ans;
}

vector<vector<int> > fourSum(vector<int> &num, int target)
{
    sort(num.begin(), num.end());
    return kSum(num, 0, target, 4);
}
```
