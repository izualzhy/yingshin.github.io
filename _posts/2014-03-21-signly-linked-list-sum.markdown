---
title:  "单链表之和"
date:   2014-03-21 17:23:03
tags: list  interview
---

[@陈利人](http://weibo.com/p/1005051915548291/home?from=page_100505&mod=TAB#place)在微博上发布了很多有意思的题目，建议大家关注下。  
题目地址在[这里](http://weibo.com/1915548291/zCToz2NYS)。
> 两个单链表（singly linked list），每一个节点里面一个0-9的数字， 输入就相当于两个大数了。然后返回这两个数的和（一个新list）。这两个输入的list 长度相等。 要求是：1. 不用递归。2. 要求算法在最好的情况下，只遍历两个list一次， 最差的情况下两遍。

解答：  
> 第一遍遍历两个链表相加并且存储，第二遍遍历处理节点*data >= 10*的情况。  
> 如果节点值>10,需要向前进位,需要两个指针，第一个不断迭代链表称为迭代指针，第二个记录需要进位到的节点称为位置指针，此时考虑两种情况：
> * 前一个节点值<9,则前一个节点值+1即可
> * 前一个节点值==9,则需要向前进位直到节点值<9,中间的节点值变0,截止的节点值+1。因此处理这一步需要记录截止的那个节点的位置。
> 对上述两种情况，如果向前遇到链表头，需要更新链表头
> 如果节点值==9,往下一个位置迭代，此时不更新位置指针，因为如果迭代指针需要向前进位，进位到不为9的位置指针才行。
> 如果节点值<9,往下一个位置迭代，同时还需要一个指针记录当前位置。

<!--more-->

代码如下:

```
/*
 * =====================================================================================
 *       Filename:  singly_linked_list_sum.cpp
 *    Description:  两个单链表（singly linked list），每一个节点里面一个0-9的数字，输入就相当于两个大数了。
 *                  然后返回这两个数的和（一个新list）。这两个输入的list长度相等。
 *                  要求是：
 *                  不用递归；
 *                  要求算法在最好的情况下，只遍历两个list一次 ，最差的情况下两遍。
 * =====================================================================================
 */
#include <stdlib.h>
#include <string.h>
#include <iostream>
#include <iomanip>
using namespace std;

//Node Type
struct Node
{
    Node(int i = -1, Node* n = NULL)
        : data(i)
        , next(n)
    {}
    int data;
    Node* next;
};

//global nodes pool.
Node gNodes[100];
int gIndex = 0;

void Init()
{
    memset(gNodes, 0, sizeof(gNodes));
    gIndex = 0;
}

//global nodes factory.
Node* ProduceNode(int i = -1, Node* next = NULL)
{
    gNodes[gIndex].data = i;
    gNodes[gIndex].next = next;
    return &gNodes[gIndex++];
}

//convert num to node list,return the listhead.
Node* Num2Node(long long num)
{
    Node* pre = NULL;
    while (num)
    {
        pre = ProduceNode(num % 10, pre);
        num /= 10;
    }

    return pre;
}

//convert node list to num and return.
long long Node2Num(Node* node)
{
    long long res = 0;
    while (node)
    {
        res = res * 10 + node->data;
        node = node->next;
    }

    return res;
}

//get the list sum, stored in a new list, and return the list head.
Node* SumNode(Node* a, Node* b)
{
    Node* pA = a, *pB = b, *pre = NULL, *head = NULL;
    while (pA && pB)
    {
        Node* node = ProduceNode(pA->data + pB->data);

        head = head ? head : node;
        if (pre)
            pre->next = node;
        pre = node;
        pA = pA->next;
        pB = pB->next;
    }

    //deal with data >= 10
    Node* p = NULL, *q = head;
    while (q)
    {
        if (q->data < 9)
        {
            p = q;
        }
        else if (q->data == 9)
        {
        }
        //需要进位
        if (q->data >= 10)
        {
            q->data %= 10;
            if (p)
            {
                p->data++;
                p = p->next;
                while (p != q)
                {
                    p->data = 0;
                    p = p->next;
                }
            }
            else
            {
                Node* node = ProduceNode(1, head);
                while (head != q)
                {
                    head->data = 0;
                    head = head->next;
                }
                head = node;
            }
            p = q;
        }
        q = q->next;
    }

    return head;
}

//Test Function
void RandomTest()
{
    for (int i = 0; i < 100; ++i)
    {
        Init();
        int count = abs(rand())%10 + 1;
        long long a = 0, b = 0;
        for (int j = 0; j < count; ++j)
        {
            if (j == 0)
            {
                a = abs(rand())%9 + 1;
                b = abs(rand())%9 + 1;
            }
            else
            {
                a = a*10 + abs(rand())%10;
                b = b*10 + abs(rand())%10;
            }
        }
        cout << setw(15) << a << " "
             << setw(15) << b << " " 
             << setw(15) << (a + b == Node2Num(SumNode(Num2Node(a), Num2Node(b)))) << endl;
    }
}

int main()
{
    long a = 123456789, b = 987654321;
    Init();
    cout << (a + b == Node2Num(SumNode(Num2Node(a), Num2Node(b)))) << endl;
    Init();
    cout << (a + a == Node2Num(SumNode(Num2Node(a), Num2Node(a)))) << endl;
    Init();
    cout << (b + b == Node2Num(SumNode(Num2Node(b), Num2Node(b)))) << endl;

    RandomTest();
    return 0;
}
```
