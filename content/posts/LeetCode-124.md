---
title: "LeetCode 124 之 二叉树的最大路径和"
date: 2019-05-22T23:14:47+08:00
draft: false
---

# 124. Binary Tree Maximum Path Sum

**难度**：Hard

**题目链接**：https://leetcode.com/problems/binary-tree-maximum-path-sum/

**题目描述**：

```
Given a non-empty binary tree, find the maximum path sum.

For this problem, a path is defined as any sequence of nodes from some starting node to any node in the tree along the parent-child connections. The path must contain at least one node and does not need to go through the root.

Example 1:

Input: [1,2,3]

       1
      / \
     2   3

Output: 6
Example 2:

Input: [-10,9,20,null,null,15,7]

   -10
   / \
  9  20
    /  \
   15   7

Output: 42
```

**思路**：

这里面有一个默认的事实，那就是不管这个路径在哪，他都肯定有一个最高的节点。我们不妨可以设这个节点为A，以A为最高节点的路径的最大长度是多少呢？

A有左右两个孩子（当然孩子可以是空），以每个孩子为起点，向下延伸，可以得到很多条单向的路径，这其中当然有一个最大路径。我们将以左孩子为起点的最大路径的值记为left_val，将以右孩子为起点的最大路径的值记为right_val，显然以A为最高点的最大路径只可能有以下四种情况：

1. 若left_val < 0, 且right_val < 0, 那最大路径为A节点本身：maxPathSum(A) = A.val
2. 若left_val > 0, 且right_val < 0, 那最大路径为A节点和以A的左孩子为起点的最大路径：maxPathSum(A) = A.val + left_val
3. 若left_val < 0, 且right_val > 0, 那最大路径为A节点和以A的右孩子为起点的最大路径：maxPathSum(A) = A.val + right_val
4. 若left_val > 0, 且right_val > 0, 那最大路径为A节点和以A的左、右孩子为起点的最大路径三者的联合：maxPathSum(A) = A.val + left_val + right_val

综上，以A为最高点的最大路径是上面四种情况的最大值。那我们遍历二叉树的所有节点，求以每个节点为最高节点的最大路径的最大值即可。

**代码**：

```java
public class Solution {
    int max = Integer.MIN_VALUE;
    public int maxPathSum(TreeNode root) {
        dp(root);
        return max;
    }

    public int dp(TreeNode root) {
        if (root == null) return 0;
        int left = dp(root.left);
        int right = dp(root.right);
        left = Math.max(0, left);
        right = Math.max(0, right);
        // 跟一般的题目不一样的是
        // 这里max的计算方式和return返回的不是一个值
        // max计算的是左右两条路径和的最大值
        max = Math.max(left + right + root.val, max);
        // return的是单边的路径
        return Math.max(left, right) + root.val;
    }
}
```

**时间复杂度**：$O(n)$

**空间复杂度**：$O(\log n)$

思路参考自：[二叉树中的最大路径和](https://blog.csdn.net/guoziqing506/article/details/51745846)
