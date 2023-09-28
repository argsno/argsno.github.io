---
title: "LeetCode 206 -- Reverse Linked List"
date: 2019-05-26T01:02:55+08:00
draft: false
tags:
  - LeetCode
---

难度：Easy
## 题目要求
```
Reverse a singly linked list.

Example:

Input: 1->2->3->4->5->NULL
Output: 5->4->3->2->1->NULL
Follow up:

A linked list can be reversed either iteratively or recursively. Could you implement both?
```
## 中文说明
反转一个单链表。
## 题解
### 迭代法
通过维护三个指针，分别指向prev、cur、next，循环遍历每个节点时，对其进行翻转。注意设置初始的prev为null。

```java
public class Solution {
    public ListNode reverseList(ListNode head) {
        ListNode prev, cur, next;
        prev = null;
        cur = head;
        while (cur != null) {
            next = cur.next;
            cur.next = prev;
            prev = cur;
            cur = next;
        }
        return prev;
    }
}
```
时间复杂度：O(N)

空间复杂度：O(1)

### 递归法

```java
class Solution {
    public ListNode reverseList(ListNode head) {
        return helper(head, null);
    }

    private ListNode helper(ListNode head, ListNode newHead) {
        if (head == null) return newHead;
        ListNode next = head.next;
        head.next = newHead;
        return helper(next, head);
    }
}
```
- 时间复杂度：O(N)
- 空间复杂度：O(1)（尾递归）