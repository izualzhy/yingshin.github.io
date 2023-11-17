---
title:  "非递归遍历二叉树"
date:   2014-03-17 18:19:03
excerpt: "采用非递归方法前序，中序，后序遍历二叉树"
tags: algorithm
---

遍历二叉树算是基础知识了，看到leetcode上有这个题目，就做了一下。  
分别是前序，中序，后序遍历。  
其中前序，中序非常近似，后序稍微麻烦些。  
地址分别在这里

* [Given a binary tree, return the preorder traversal of its nodes' values.](http://oj.leetcode.com/problems/binary-tree-preorder-traversal/)
* [Given a binary tree, return the inorder traversal of its nodes' values.](http://oj.leetcode.com/problems/binary-tree-inorder-traversal/)
* [Given a binary tree, return the postorder traversal of its nodes' values.](http://oj.leetcode.com/problems/binary-tree-postorder-traversal/)


结合程序简单分析一下

<!--more-->

前序遍历：  

访问根节点后，不断向左递归直到左子树为空，此时左子树便利完毕。
找到栈上最上面的节点，也就是刚刚访问的结点，此时是第二次访问根节点，开始遍历右子树即可。

```
vector<int> PreOrderTraversal(TreeNode *root)
{
    stack<TreeNode*> s;
    vector<int> v;
    while (root || !s.empty())
    {
        if (root)
        {
            v.push_back(root->val);
            s.push(root);
            root = root->left;
        }
        else
        {
            TreeNode* node = s.top();
            s.pop();
            root = node->right;
        }
    }

    return v;
}
```

中序版本非常相似，不过是在弹出时访问节点，因为弹出时表明左子树已经便利完毕。

```
vector<int> InOrderTraversal(TreeNode *root)
{
    vector<int> v;
    while (root || !s.empty())
    {
        if (root)
        {
            s.push(root);
            root = root->left;
        }
        else
        {
            TreeNode* node = s.top();
            v.push_back(node->val);
            s.pop();
            root = node->right;
        }
    }

    return v;
}
```

后序便利比较麻烦的地方在于，第一次遇到根节点时应当先访问左子树，第二次碰到时需要访问右子树，第三次才访问自身节点并且弹出。  
因此设置了preNode来区分第二次和第三次访问，preNode记录上次弹出的节点，如果与栈顶节点右子结点相同则表示第三次访问。  
切换到右子树时置为空，因为切换到右子树会弹入新的结点，相当于一颗新的树（赋值为初始值），用于判断右子树的左子树遍历完毕。  
这一部分可能有点绕，最好用gdb实际观测一下。  

```
vector<int> PostOrderTraversal(TreeNode* root)
{
    stack<TreeNode*> s;
    vector<int> v;
    TreeNode* preNode = NULL;
    while (root || !s.empty())
    {
        if (root)
        {
            s.push(root);
            root = root->left;
        }
        //右子树遍历完毕
        else if (preNode == s.top()->right)
        {
            preNode = s.top();
            v.push_back(preNode->val);
            s.pop();
        }
        //左子树遍历完毕
        else
        {
            TreeNode* node = s.top();
            root = node->right;
            preNode = NULL;
        }
    }

    return v;
}
```
