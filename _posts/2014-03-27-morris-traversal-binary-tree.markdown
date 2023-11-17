---
title:  "Morris遍历二叉树"
date:   2014-03-26 14:36:03
tags: algorithm
---

Morris遍历不使用栈，O(1)空间进行二叉树的遍历。  
原理就是：  
> 利用叶子结点的右空指针，临时性的指向中序遍历的后继结点。    

与*[这篇文章](http://izualzhy.cn/traversal-binary-tree-nonrecursively/)*的想法类似，前序和中序非常相近，顺序是相同的，不同的是访问的时机。  

### 1. 前序、中序遍历  

```
#pseudo code

while 结点存在:
    if 左子树存在:
        找到左子树的最右结点，
        如果发现左子树的最右结点:#这个最右结点也就是当前结点在中序遍历下的前驱结点
            该结点右指针指向当前结点
        否则如果最右结点的右指针已经指向当前结点:
            还原最右结点的右指针指向NULL
    else:
        访问该结点
        转到右孩子结点
```  

<!--more-->

Morris遍历与利用栈遍历都是非递归的，不过不同的是空间复杂度由O(n)&rarr;O(1)。    
对任意一个根结点，实际上是有两次经过的，存在左子树的情况下：   
* 第一次修改其*左子树的最右指针的右指针*指向自身  
* 第一次还原其*左子树的最右指针的右指针*指向NULL    
判断第几次经过该结点的条件就是看上述指针的指向是自身还是NULL。  

注意这种访问顺序是由左子树以及左子树的最右结点返回到自身(root)的顺序。  
对于中序、前序遍历，访问顺序相同，不同的是访问时机。  


1. 前序两次访问的时间：  
1.1 没有左子树遍历右子树之前   
1.2. 有左子树第一次遍历到根节点,下一步会遍历左子树。      
2. 中序两次访问的时间：   
2.1. 没有左子树遍历右子树之前   
2.2. 有左子树第二次遍历到根节点，表示遍历左子树已经返回。   


代码如下:

```
void MorrisPreorder(Node* node)
{
    while (node)
    {
        //有左子树，判断是第几次经过
        if (node->left)
        {
            Node* temp = node->left;
            while (temp->right && temp->right != node)
                temp = temp->right;
            //最右子节点，即node的中序前驱结点，指向node。
            if (!temp->right)
            {
                temp->right = node;
                cout << node->val;
                node = node->left;
            }
            //已经指向了node
            else
            {
                temp->right = NULL;//恢复前驱结点的指向为NULL
                node = node->right;//遍历右子树
            }
        }
        //没有左子树的情况
        else
        {
            cout << node->val;
            node = node->right;
        }
    }

    cout <<endl;
}

void MorrisInorder(Node* node)
{
    while (node)
    {
        if (node->left)
        {
            Node* temp = node->left;
            while (temp->right && temp->right != node)
                temp = temp->right;
            if (!temp->right)
            {
                temp->right = node;
                node = node->left;
            }
            else
            {
                cout << node->val;
                temp->right = NULL;
                node = node->right;
            }
        }
        else
        {
            cout << node->val;
            node = node->right;
        }
    }
    cout << endl;
}
```

### 2. 后序遍历  
跟使用栈遍历的方法类似，后序会比较麻烦些。  
先说下步骤，再说下我的理解。  
注意:   
后序的指针指向并没有变，不同的是输出方式。前序、中序只是输出位置不同，后序变化较大，需要反转某个范围内的指针right指向。   

```
#pseudo code

建立一个临时结点dump,令其左孩子是root。   
当前结点设置为临时结点dump。  
while 结点存在:
    if 左子树存在:
        找到左子树的最右结点，
        如果发现左子树的最右结点:#这个最右结点也就是当前结点在中序遍历下的前驱结点
            该结点右指针指向当前结点
        否则如果最右结点的右指针已经指向当前结点:
            倒序输出当前结点的左孩子到该右指针这条路径上的所有结点。
    else 
        转到右孩子结点

```

倒序输出的序列也就是：  
**从左子树的最右结点到左孩子本身。即输出了`LRN`里的`RN`。**    
至于为什么使用dump,我们可以这么理解：    
> Morris是利用了叶子结点的right指针，该叶子结点在二叉树中序遍历里是某个子树root的前驱结点。  
> 当访问到该叶子结点时，通过事先修改过的right指针可以回溯到某个父节点。   
> 只要是左孩子，无论是右子树的左孩子还是左子树的左孩子，都可以回溯到父节点。  
> 对root而言，左子树的最右孩子right指针会指向自己。可以回溯。  
> 但如果当前为最右子树的最右结点，则无法回溯到root，因为这个叶子结点是中序遍历的最后一个结点。  
> 不存在前驱结点。也就是会一直循环上述伪代码里的else部分。  
> 因此下面的程序如果直接输出root本身的话，root到该最右结点这条路径是无法输出的。        
> 也就是会正确的输出整个左子树。  
> 一个比较直观的办法就是，虚拟一棵树，该树左子树为原来的树。  
> 即
> 
> ```
> 建立一个临时结点dump,令其左孩子是root。   
> 当前结点设置为临时结点dump。  
> ```   
> 当然基于以上分析，不使用dump，单独输出一遍最右路径也是可以的。（已提交并在leetcode上测试通过。）

代码如下：



```
//反转的代码
void Reverse(Node* start, Node* end)
{
    Node* pre = start;
    Node* cur = start->right;

    while (pre != end)
    {
        Node* next = cur->right;
        cur->right = pre;
        pre = cur;
        cur = next;
    }
}
```



```   
//后序遍历版本一   
void MorrisPostorder(Node* node)
{
    Node dump;
    dump.left = node;
    node = &dump;
    while (node)
    {
        if (!node->left)
        {
            node = node->right;
        }
        else
        {
            Node* pre = node->left;
            while (pre->right && pre->right != node)
                pre = pre->right;

            if (pre->right)
            {
                Reverse(node->left, pre);
                Node* temp = pre;
                while (temp != node->left)
                {
                    cout << temp->val;
                    temp = temp->right;
                }
                cout << temp->val;
                Reverse(pre , node->left);
                node = node->right;
                pre->right = NULL;
            }
            else
            {
                pre->right = node;
                node = node->left;
            }
        }
    }
    cout << endl;
}
```

不使用dump:  


```
//后序遍历版本二
void MorrisPostorder2(Node* node)
{
    Node* root = node;
    while (node)
    {
        if (!node->left)
        {
            node = node->right;
        }
        else
        {
            Node* pre = node->left;
            while (pre->right && pre->right != node)
                pre = pre->right;

            if (pre->right)
            {
                Reverse(node->left, pre);
                Node* temp = pre;
                while (temp != node->left)
                {
                    cout << temp->val;
                    temp = temp->right;
                }
                cout << temp->val;
                Reverse(pre , node->left);
                node = node->right;
                pre->right = NULL;
            }
            else
            {
                pre->right = node;
                node = node->left;
            }
        }
    }

    Node* rightMost = root;
    while (rightMost && rightMost->right)
        rightMost = rightMost->right;
    Reverse(root, rightMost);

    Node* temp = rightMost;
    while (temp != root)
    {
        cout << temp->val;
        temp = temp->right;
    }
    cout << temp->val;
    Reverse(rightMost, root);
    rightMost->right = NULL;//注意最右结点的最右指针要重新设置为NULL。

    cout << endl;
}
```

可以参考下这篇文章    
[Morris Traversal方法遍历二叉树（非递归，不用栈，O1空间](http://www.cnblogs.com/AnnieKim/archive/2013/06/15/MorrisTraversal.html)。
在陈利人的微博里也有过介绍。  
发现二叉树的遍历很多有意思的可分析的地方，欢迎讨论。 
