---
layout: post
title: "[golang/std] 2 - container/heap"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - golang
  - std
  - programming
---

golang里要使用优先队列略微麻烦，需要自己实现 `heap.Interface` 下的几个接口，比起 c++ 的容器使用起来更麻烦一些。

```
type rank struct {
	score, index int
}

type priorityQueue []rank

func (q *priorityQueue) Push(x interface{}) {
	*q = append(*q, x.(rank)) // note: 需要赋值给指针否则无法更新实际的heap
}

func (q *priorityQueue) Pop() interface{} {
	pq := *q
	size := len(pq)
	x := pq[size - 1]
	*q = pq[:size - 1]  // 同上
	return x
}

func (q *priorityQueue) Less(i, j int) bool {
	pq := *q
	return pq[i].score > pq[j].score
}

func (q *priorityQueue) Swap(i, j int) {
	pq := *q
	pq[i], pq[j] = pq[j], pq[i]
}

func (q *priorityQueue) Len() int {
	return len(*q)
}
```

默认是最小堆（如果`Less()`函数是小于的话)，上面实现中为大于，表示实现的是按照 `rank.score` 排序的最大堆。另外要记得 c++ 中默认是最大堆。

调用示例：
```
func TestPQ(t *testing.T) {
	score := []int{1,5,12,6,4,8,36,20}
	pq := priorityQueue{}
	for i := range score {
		pq = append(pq, rank{score: score[i], index: i})
	}
	heap.Init(&pq)
	for pq.Len() > 0 {
		t.Log(heap.Pop(&pq))
	}
}
```

The End

---------