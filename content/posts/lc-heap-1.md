---
title: 淺談 Heap - 1. 介紹與實作
date: 2022-02-12 00:00:00
draft: false
tags:
  - LeetCode
categories:
  - LeetCode
---

## 淺談 Heap 資料結構

### 前言

這個系列會紀錄我在 [Leetcode](https://leetcode.com/) 上複習的一些常見的東西。基本上會跟著探索卡的進度走。雖然這系列應該早就在課堂上學會了，但我就豆腐腦，雖然已經理解了，但是只要一過一段時間，記憶就這樣被回收了，我都懷疑我腦袋有一個GC回收器...

### 優先佇列 (Priority Queue)

再講 Heap 之前，要先介紹一下優先佇列 Priority Queue：

既然優先佇列的名稱裡都有佇列了，那他肯定跟佇列有關係對吧？

優先佇列就像是普通的先進先出(FIFO)的佇列一樣，但是其每個元素都有優先權(Priority)這個變數，優先佇列的特點就是：**優先權較高者的永遠會先被處理**，如果對 Heap 有印象的人，看到現在應該知道為什麼要先介紹優先佇列了。

那要怎麼實現 Priority Queue 呢？

Priority Queue 就如同佇列一樣有兩個關鍵方法：`push()`、`pull()`

如果將元素存放在一個未排序列表中， 那`push()` 操作正常來說只需要`O(1)`就能完成，因為只是單純的將元素推到這個列表的最後端，範例如下：

```go
func push(n *Node) {
    unsortedList = append(unsortedList, n)
}
```

但是`pull()`操作呢？

正常來說需要`O(n)`時間，也就是需要遍歷完整個列表，範例如下：

```go
func pull() *Node {
    if len(unsortedList) == 0 {
        return nil
    }
    
    maxIndex, highest := 0, unsortedList[0]
    for i := 1; i < len(unsortedList); i++ {
        if highest.Priority < unsortedList[i].Priority {
            highest = unsortedList[i]
            maxIndex = i
        }
    }
    // remove highest item
    unsortedList = append(unsortedList[:maxIndex], unsortedList[maxIndex + 1:]...)
    return highest
}
```

如果反過來將他保存在已排序的列表中，那`push()`操作需要花`O(n)`時間，`pull()`操作僅需`O(1)`時間。

優化時間複雜度我們可以使用**Heap**這個資料結構來做，這樣就可以使我們的`push()`操作跟`pull()`操作都僅需`O(logn)`。

## Heap

前面講了那麼多，那 Heap 是什麼呢？

- **Heap 是一個完全二元樹(complete binary tree)，每個節點都必須大於(或小於)其子節點。**

- **插入及刪除操作的時間複雜度為：`O(logn)`**

- **尋找最大/最小值的時間複雜度為：`O(1)`**

Heap 主要有分兩種：**最大堆(Max-Heap)、最小堆(Min-Heap)**。

![最小堆、最大堆範例](/img/heap/min-max-heap.png)

Heap 通常會使用 **陣列** 實作而不是傳統的鏈表(linked list)，因為陣列恰巧滿足完全二元樹的定義：**節點由上至下、由左至右排列**，而且相對於鏈表來說，陣列在記憶體中是連續的，所以存取元素相對快速，內存效率較高，並且隨機訪問的時間複雜度為`O(1)`。

### Heap 插入操作

Heap 插入就只分兩個步驟，第一步插入元素，第二步比較子元素跟父元素之間的大小，若子元素較小/大，則交換，交換到子元素在合適的位置。

### Heap 刪除操作

在 Heap 中，刪除操作就是刪除堆頂的元素，這時候你需要將最尾端的元素跟最前端的元素交換，刪除最尾端的元素，然後將堆頂的元素往下移動到合適的位置即可

### 實作

```go
package main

import (
	"fmt"
)

type Heap struct {
	data []int
}

func NewHeap() *Heap {
	return &Heap{data: []int{}}
}

func (h *Heap) Insert(val int) {
	h.data = append(h.data, val)
	// 取得當前節點位置
	current := len(h.data) - 1
	for {
		// 取得父節點位置
        parent := (current - 1) / 2
        if parent == current || h.data[parent] < h.data[current] {
            break
        }
		h.data[parent], h.data[current] = h.data[current], h.data[parent]
		current = parent
	}
}

func (h *Heap) Deletion() int {
	if len(h.data) == 0 {
		return -1
	}

	// 取出堆頂元素
	top := h.data[0]
	// 將最後一個元素放到堆頂
	h.data[0] = h.data[len(h.data)-1]
	// 刪除最後一個元素
	h.data = h.data[:len(h.data)-1]
	// 將堆頂元素往下移動
	current := 0
	for {
		// 取得左右子節點的位置
		left := current * 2 + 1
		right := current * 2 + 2
		// 如果左節點存在、右節點不存在，且左節點小於堆頂，就交換
		if left < len(h.data) && right >= len(h.data) && h.data[left] < h.data[current] {
			h.data[left], h.data[current] = h.data[current], h.data[left]
			current = left
		// 如果右節點存在、左節點不存在，且右節點小於堆頂，就交換
		} else if left >= len(h.data) && right < len(h.data) && h.data[right] < h.data[current] {
			h.data[right], h.data[current] = h.data[current], h.data[right]
			current = right
		// 如果兩者都存在，且兩者至少有一個小於堆頂，就跟左右節點比較小的那個交換
		} else if left < len(h.data) && right < len(h.data) {
			if h.data[left] < h.data[current] || h.data[right] < h.data[current] {
				if h.data[left] < h.data[right] {
					h.data[left], h.data[current] = h.data[current], h.data[left]
					current = left
				} else {
					h.data[right], h.data[current] = h.data[current], h.data[right]
					current = right
				}
			} else {
				break
			}
		// 若兩者都不存在，就跳出迴圈
		} else {
			break
		}
	}
	return top
}

func (h *Heap) Print(index, level int) {
	if index > len(h.data) - 1 {
		return
	}
	fromIndex := level - 1
	for i := 0; i < level; i++ {
		if i == fromIndex {
			fmt.Print("├── ")
		} else {
			fmt.Print("│   ")
		}
	}
	fmt.Printf("%d\n", h.data[index])
	h.Print(2 * index + 1, level + 1)
	h.Print(2 * index + 2, level + 1)
}

func main() {
	h := NewHeap()
	h.Insert(6)
	h.Insert(9)
	h.Insert(10)
	h.Insert(15)
	h.Insert(12)
	h.Insert(13)
	h.Print(0, 0)
	fmt.Println()

	h.Deletion()
	h.Print(0, 0)
	fmt.Println()

	h.Deletion()
	h.Print(0, 0)
	fmt.Println()

	h.Deletion()
	h.Print(0, 0)
	fmt.Println()
}
```

### 結果
```
6
├── 9
│   ├── 15
│   ├── 12
├── 10
│   ├── 13

9
├── 12
│   ├── 15
│   ├── 13
├── 10

10
├── 12
│   ├── 15
├── 13

12
├── 15
├── 13
```
