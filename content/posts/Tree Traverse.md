---
title: "树的遍历"
date: 2019-04-23T15:17:12+08:00
draft: false
---

二叉树的遍历（特别是二查查找树）是数据结构与算法的基础知识，主要的遍历算法有四种，包括了先序遍历、中序遍历、后序遍历，还有层序遍历。不同的遍历算法流程经常会搞混了，特别是面试的时候。这里进行总结一下。

## 前序遍历(Pre-Order Traversal)

指先访问根，然后访问子树的遍历方式。
```c
void pre_order_traversal(TreeNode *root) {
    // Do Something with root
    if (root->lchild != NULL)
        pre_order_traversal(root->lchild);
    if (root->rchild != NULL)
        pre_order_traversal(root->rchild);
}
```

## 中序遍历(In-Order Traversal)
指先访问左（右）子树，然后访问根，最后访问右（左）子树的遍历方式
```c
void in_order_traversal(TreeNode *root) {
    if (root->lchild != NULL)
        in_order_traversal(root->lchild);
    // Do Something with root
    if (root->rchild != NULL)
        in_order_traversal(root->rchild);
}
```

## 后序遍历(Post-Order Traversal)
指先访问子树，然后访问根的遍历方式
```c
void post_order_traversal(TreeNode *root) {
    if (root->lchild != NULL)
        post_order_traversal(root->lchild);
    if (root->rchild != NULL)
        post_order_traversal(root->rchild);
    // Do Something with root
}
```

## 层次遍历
层次遍历又称为二叉树的广度优先遍历，会先访问离根节点最近的节点，一层一层往下遍历。算法借助队列实现。
```c
void Layer_Traver(tree *root) {
    int head = 0, tail = 0;
    tree *p[SIZE] = {NULL};
    tree *tmp;
    if (root != NULL) {
        p[head] = root;
        tail++;
        // Do Something with p[head]
    } else
        return;
    while (head < tail) {
        tmp = p[head];
        // Do Something with p[head]
        if (tmp->left != NULL) { // left
            p[tail] = tmp->left;
            tail++;
        }
        if (tmp->right != NULL) { // right
            p[tail] = tmp->right;
            tail++;
        }
        head++;
    }
    return;
}
```

## 深度优先遍历
先访问根结点，后选择一子结点访问并访问该节点的子结点，持续深入后再依序访问其他子树，可以轻易用递回或栈的方式实现。
```c
void travel(treenode* nd){
    for(treenode* nex :　nd->childs){ // childs 存放指向其每個子結點的指標
        travel(nex);   
    }
    return;
}
```