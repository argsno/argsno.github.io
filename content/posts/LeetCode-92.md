---
title: "LeetCode 92 -- Reverse Linked List II"
date: 2019-05-26T01:05:27+08:00
draft: false
tags:
  - LeetCode
---

难度：Medium
## 题目要求
```
Reverse a linked list from position m to n. Do it in one-pass.

Note: 1 ≤ m ≤ n ≤ length of list.

Example:

Input: 1->2->3->4->5->NULL, m = 2, n = 4
Output: 1->4->3->2->5->NULL
```
## 中文说明
```
反转从位置 m 到 n 的链表。请使用一趟扫描完成反转。

说明:
1 ≤ m ≤ n ≤ 链表长度。
```
## 题解
维护四个指针：dummy、prev、start、then
- dummy：头节点
- prev：指向翻转链表的前一个节点
- start：指向翻转链表的第一个节点的位置
- then：指向下一个准备翻转的节点
```java
public class Solution {
    public ListNode reverseBetween(ListNode head, int m, int n) {
        ListNode dummy = new ListNode(0);
        dummy.next = head;
        ListNode prev, start, then;
        prev = dummy;
        for (int i = 0; i < m - 1; i++) prev = prev.next;
        start = prev.next;
        then = start.next;

        for (int i = 0; i < n - m; i++) {
            start.next = then.next;
            then.next = prev.next;
            prev.next = then;
            then = start.next;
        }
        return dummy.next;
    }
}
```
- 时间复杂度：O(N)
- 空间复杂度：O(1)