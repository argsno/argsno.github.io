---
title: "现代化的缓存设计方案"
date: 2019-03-26T17:18:41+08:00
draft: true
---

> 原文地址：[Design Of A Modern Cache—Part Deux](http://highscalability.com/blog/2019/2/25/design-of-a-modern-cachepart-deux.html)

在之前的[文章](http://highscalability.com/blog/2016/1/25/design-of-a-modern-cache.html)里，主要介绍了[Caffeine](https://github.com/ben-manes/caffeine)使用的缓存算法，特别是其中的驱逐策略以及并发模型。自从上次文章发布之后，我们又对驱逐算法进行了更新并探索了另一种过期的机制。

## 驱逐策略

[Window TinyLFU](https://dl.acm.org/citation.cfm?id=3149371) (W-TinyLFU)

## 过期机制

由于之前对于过期更高级的支持尚处于开发中，因此仅仅只是简要的提及。典型的做法要么是使用一个O(lg n)的优先级队列维持有序顺序，要么是将无用的条目保存在缓存中，并依赖于缓存的最大大小策略来最终驱逐出去。Caffeine仅仅使用平摊时间O(1)的算法，