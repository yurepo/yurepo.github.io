---
title: "Leetcode 1632"
date: 2021-08-08T23:53:41+08:00
draft: false
categories:
    - LeetCode
    - golang
---

# Rank Transform of a Matrix

最近在刷 LC 的時候，看到這題很有趣的題目，這題的其中一個解法使用了**併查集(Disjoint-set data structure)**，下面來回顧一下併查集的概念。

## 併查集

**併查集**用於處理一些不交集的合併與查詢問題，最常用的一種實現為**不交集森林**，其空間複雜度(SC)以及操作時間複雜度(TC)如下：

操作時間複雜度:
$$
Time\hspace*{2mm}Complexity = O(\alpha(n))
$$

空間複雜度：
$$
Space\hspace*{2mm}Complexity = O(n)
$$

### 不交集森林

不交集森林把每個集合以樹狀表示，每個元素都儲存了回到父元素的值，根元素則儲存回到自身的值或是無效值。

使用不交集森林實現的併查集，每個集合的代表元素為根(Root)元素。

### 操作簡介

* **添加**：將元素
* **查詢**：找尋該元素所在的集合
* **合併**：將兩個元素的集合合併在一起