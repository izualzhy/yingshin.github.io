---
title:  "如何写出正确的二分法以及分析"
date: 2014-04-18 16:43:18
excerpt: "binary search problems analysis."
tags: algorithm
---

二分法在平时经常用到，除了查找某个key的下标以外，还有很多变形的形式。  
比如STL里的`lower_bound,upper_bound`。  

总结一下的注意点有这么几个：   
  1. 数组是非递增还是非递减   
  2. 结束条件，即while (condition) 应当是<还是<=   
  3. 求mid应当是偏向左还是右，即 mid = (left + right) >> 1, 还是 mid = (left + right + 1) >> 1    
  4. 如何得到循环不变式   
  5. while结束后是否需要判断一次条件**    


整理了下常见的几个问题如下：  

1. [查找值key的下标，如果不存在返回-1.](#id1)    
2. [查找值key第一次出现的下标x，如果不存在返回-1.](#id2)    
3. [查找值key最后一次出现的下标x，如果不存在返回-1.](#id3)    
4. [查找刚好小于key的元素下标x，如果不存在返回-1.](#id4)    
5. [查找刚好大于key的元素下标x，如果不存在返回-1,等价于std::upper\_bound.](#id5)    
6. [查找第一个>=key的下标，如果不存在返回-1,等价于std::lower\_bound.](#id6)     

[leetcode](http://oj.leetcode.com/problems)上也有很多类似的题目。   
例如：[Search a 2D Matrix](http://oj.leetcode.com/problems/search-a-2d-matrix/)

二分查找，必须条件是有序数组，然后不断折半，几乎每次循环都可以降低一半左右的数据量。  
因此是O(lgN)的方法，要注意的是二分查找要能够退出，不能陷入死循环。  

二分查找用到的一个重要定义就是循环不变式，顾名思义，就是在循环中不会改变这么一个性质。  
举个例子，插入排序，不断的循环到新的索引，但保持前面的排序性质不变。  
其实就是数学归纳法，具体的定义不用管，我们在第一个例子里看下。  

<!--more-->

#### <a name="id1" id="id1">1. 查找值为key的下标，如果不存在返回-1.</a>   

先看一下伪代码:

```
while left <= right:
    mid = (left + right) >> 1
    if array[mid] > key:
        right = mid - 1
    else if array[mid] < key:
        left = mid + 1
    else
        return mid
return -1
```

这里面包含怎样的循环不变式呢？  
**如果中间值比key大，那么[mid, right]的值我们都可以忽略掉了，这些值都比key要大。  
  只要在[left, mid-1]里查找就是了。相反，如果中间值比key小，那么[left, mid]的值可以
  忽略掉，这些值都比key要小，只要在[mid+1, right]里查找就可以了。如果相等，表示找到了，
  可以直接返回。因此，循环不变式就是在每次循环里，我们都可以保证要找的index在我们新构造
  的区间里。如果最后这个区间没有，那么就确实是没有**    
注意mid的求法，可能会int越界，但我们先不用考虑这个问题，要记住的是这点：     
**mid是偏向left的，即如果left=1,right=2,则mid=1。**

参考代码：  

```
int BS(const VecInt& vec, int key)
{
    int left = 0, right = vec.size() - 1;
    while (left <= right)
    {
        int mid = (left + right) >> 1;
        if (vec[mid] > key)
            right = mid - 1;
        else if (vec[mid] < key)
            left = mid + 1;
        else
            return mid;
    }

    return -1;
}
```

接着考虑一个复杂一点的问题

#### <a name="id2" id="id2">2. 查找值key第一次出现的下标x，如果不存在返回-1.</a>


我们仍然考虑中间值与key的关系：  
1. 如果array[mid]&lt;key，那么x一定在[mid+1, right]区间里。  
2. 如果array[mid]&gt;key，那么x一定在[left, mid-1]区间里。   
3. 如果array[mid]&le;key，那么不能推断任何关系。  
   比如对key=1,数组{0,1,1,2,3},{0,0,0,1,2},array[mid] = array[2] &le; 1，但一个在左半区间，一个在右半区间。  
4. 如果array[mid]&ge;key，那么x一定在[left, mid]区间里。  

综合上面的结果，我们可以采用1,4即&lt;和&ge;的组合判断来实现我们的循环不变式，即  
循环过程中一直满足key在我们的区间里。  

这里需要注意两个问题：  
1. 循环能否退出,我们注意到4的区间改变里是令`right = mid`,如果left=right=mid时，循环是无法退出的。  
   换句话说，第一个问题我们始终在减小着区间，而在这个问题里，某种情况下区间是不会减小的！    
2. 循环退出后的判断问题，再看下我们的条件1,4组合，只是使得我们最后的区间满足了&ge;key，是否=key，还需要再判断一次。  

参考代码:  

```
int BS_First(const VecInt& vec, int key)
{
    int left = 0, right = vec.size() - 1;
    while (left < right)//问题1，left < right时继续，相等就break.
    {
        int mid = (left + right) >> 1;
        if (vec[mid] < key)
            left = mid + 1;
        else
            right = mid;
    }

    if (vec[left] == key)//问题2,再判断一次。
        return left;

    return -1;
}
```

接下来的这个问题还有一个小小的坑，需要注意下：   

#### <a name="id3" id="id3">3. 查找值key最后一次出现的下标x，如果不存在返回-1.</a>    
省去分析的过程，我们直接写下想到的循环不变式：  
1. 如果array[mid]&gt;key，那么x一定在[left, mid-1]区间里。  
2. 如果array[mid]&le;key, 那么x一定在[mid, right]区间里。  

这里需要注意个问题：    
在条件2里，实际上我们是令`left=mid`，但是如前面提到的，如果left=1,right=2,那么mid=left=1，   
同时又进入到条件2,left=mid=1，即使我们在while设定了`left < right`仍然无法退出循环，解决的办法很简单：  
`mid = (left + right + 1) >> 1`   
向右偏向就可以了。   

参考代码：  

```
int BS_Last(const VecInt& vec, int key)
{
    int left = 0, right = vec.size() - 1;
    while (left < right)
    {
        int mid = (left + right + 1) >> 1;
        if (vec[mid] > key)
            right = mid - 1;
        else
            left = mid;
    }
    
    if (vec[left] == key)
        return left;

    return -1;
}
```

接下来的题目都是类似的。  
只贴下循环不变式和参考代码，如果你有别的心得或者这篇文章有错误欢迎提出。  

#### <a name="id4" id="id4">4. 查找刚好小于key的元素下标x，如果不存在返回-1.</a>    
1. 如果array[mid]&lt;key，那么x在区间[mid, right]
2. 如果array[mid]&ge;key，那么x在区间[left, mid-1]

参考代码：   

```
int BS_Last_Less(const VecInt& vec, int key)
{
    int left = 0, right = vec.size() - 1;
    while (left < right)
    {
        int mid = (left + right + 1) >> 1;
        if (vec[mid] < key)
            left = mid;
        else
            right = mid - 1;
    }

    if (vec[left] < key)
        return left;

    return -1;
}
```

#### <a name="id5" id="id5">5. 查找刚好大于key的元素下标x，如果不存在返回-1,等价于std::upper\_bound.</a>    

1. 如果array[mid]&gt;key，那么x在区间[left, mid]
2. 如果array[mid]&le;key，那么x在区间[mid + 1, right]

参考代码：

```
int BS_First_Greater(const VecInt& vec, int key)
{
    int left = 0, right = vec.size() - 1;
    while (left < right)
    {
        int mid = (left + right) >> 1;
        if (vec[mid] > key)
            right = mid;
        else
            left = mid + 1;
    }

    if (vec[left] > key)
        return left;

    return -1;
}
```

#### <a name="id6" id="id6">6. 查找第一个>=key的下标，如果不存在返回-1,等价于std::lower\_bound.</a>     

1. 如果array[mid]&lt;key，那么x在区间[mid + 1, right]
2. 如果array[mid]&ge;key，那么x在区间[left, mid]

参考代码：

```
int BS_First_Greater_Or_Equal(const VecInt& vec, int key)
{
    int left = 0, right = vec.size() - 1;
    while (left < right)
    {
        int mid = (left + right) >> 1;
        if (vec[mid] < key)
            left = mid +1;
        else
            right = mid;
    }

    if (vec[left] >= key)
        return left;

    return -1;
}
```

如果有什么疑问或者错误，欢迎指出。  
最后附上完整的代码及测试：

```
/*
 * =====================================================================================
 *       Filename:  binary_search.cpp
 *    Description:  binary search.
 * =====================================================================================
 */
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <vector>
#include <algorithm>
using namespace std;

typedef vector<int> VecInt;

//print arry 
void Print(const VecInt& vec)
{
    printf("INDEX: ");
    for (unsigned int i = 0; i < vec.size(); ++i)
        printf("%3d", i);
    printf("\n");
    printf("ARRAY: ");
    for (unsigned int i = 0; i < vec.size(); ++i)
        printf("%3d", vec[i]);
    printf("\n");
}

//查找值为key的下标，如果不存在返回-1.
int BS(const VecInt& vec, int key)
{
    int left = 0, right = vec.size() - 1;
    while (left <= right)
    {
        int mid = (left + right) >> 1;
        if (vec[mid] > key)
            right = mid - 1;
        else if (vec[mid] < key)
            left = mid + 1;
        else
            return mid;
    }

    return -1;
}

//查找值key第一次出现的下标x，如果不存在返回-1.
int BS_First(const VecInt& vec, int key)
{
    int left = 0, right = vec.size() - 1;
    while (left < right)
    {
        int mid = (left + right) >> 1;
        if (vec[mid] < key)
            left = mid + 1;
        else
            right = mid;
    }

    if (vec[left] == key)
        return left;

    return -1;
}

//查找值key最后一次出现的下标x，如果不存在返回-1.
int BS_Last(const VecInt& vec, int key)
{
    int left = 0, right = vec.size() - 1;
    while (left < right)
    {
        int mid = (left + right + 1) >> 1;
        if (vec[mid] > key)
            right = mid - 1;
        else
            left = mid;
    }
    
    if (vec[left] == key)
        return left;

    return -1;
}

//查找刚好小于key的元素下标x，如果不存在返回-1.
int BS_Last_Less(const VecInt& vec, int key)
{
    int left = 0, right = vec.size() - 1;
    while (left < right)
    {
        int mid = (left + right + 1) >> 1;
        if (vec[mid] < key)
            left = mid;
        else
            right = mid - 1;
    }

    if (vec[left] < key)
        return left;

    return -1;
}

//查找刚好大于key的元素下标x，如果不存在返回-1,等价于std::upper_bound.
int BS_First_Greater(const VecInt& vec, int key)
{
    int left = 0, right = vec.size() - 1;
    while (left < right)
    {
        int mid = (left + right) >> 1;
        if (vec[mid] > key)
            right = mid;
        else
            left = mid + 1;
    }

    if (vec[left] > key)
        return left;

    return -1;
}

//查找第一个>=key的下标，如果不存在返回-1,等价于std::lower_bound.
int BS_First_Greater_Or_Equal(const VecInt& vec, int key)
{
    int left = 0, right = vec.size() - 1;
    while (left < right)
    {
        int mid = (left + right) >> 1;
        if (vec[mid] < key)
            left = mid +1;
        else
            right = mid;
    }

    if (vec[left] >= key)
        return left;

    return -1;
}

int main()
{
    srand(time(NULL));

    int count = 20;
    const int N = 10;
    while (count--)
    {
        printf("================  TEST  ================\n");
        VecInt vec;
        for (int i = 0; i < N; ++i)
            vec.push_back(rand() % N);
        sort(vec.begin(), vec.end());
        int key = rand() % N;
        Print(vec);
        printf("%20s%10d%10d\n", "Find Key:", key, BS(vec, key));
        printf("%20s%10d%10d\n", "Find First = Key:", key, BS_First(vec, key));
        printf("%20s%10d%10d\n", "Find Last = Key:", key, BS_Last(vec, key));
        printf("%20s%10d%10d\n", "Find Last < Key:", key, BS_Last_Less(vec, key));
        vector<int>::iterator iter = upper_bound(vec.begin(), vec.end(), key);
        int index = iter == vec.end() ? -1 : iter - vec.begin();
        printf("%20s%10d%10d%10d\n", "Find First > Key:", key, BS_First_Greater(vec, key), index);
        iter = lower_bound(vec.begin(), vec.end(), key);
        index = iter == vec.end() ? -1 : iter - vec.begin();
        printf("%20s%10d%10d%10d\n", "Find First >= Key:", key, BS_First_Greater_Or_Equal(vec, key), index);
    }

    return 0;
}

```

