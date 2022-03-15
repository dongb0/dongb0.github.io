---
layout: post
title: "[图论] 1 - 最小生成树"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - algorithm
  - graph
---

> 简单记录Prim和Kruskal最小生成树算法的实现思路，证明暂略

二者都贪心，Prim从已经在最小生成树顶点集合 $U$ 里，找通往不在该集合里的顶点 $V-U$ 权值最小的边，随后将其加入集合 $U$ 直到所有顶点都访问过后得到最小生成树；Kruskal按照升序遍历所有的边，如果该边连接的两个顶点不在同一个顶点集合内，则选中这条边作为最小生成树的一条边，并且将两个顶点所属的顶点集合合并为一个。

以下的代码示例来自[leetcode-1584](https://leetcode.com/problems/min-cost-to-connect-all-points/)

### Prim

优先队列，每次从队列中取一条边，要求它的from端点来自已选过的节点，to端点必须是没选过的，找 n - 1 条边(n 为顶点个数)即可停止。

复杂度：$O(Vlog(V) + Elog(V))$，其中$O(Vlog(V))$来自于从优先队列中取$V-1$条边，每次取出后更新堆的复杂度为$Vlog(V)$；$O(Elog(V))$ 来自遍历每个顶点的所有边，并将顶点 V 的权重更新为 to 端点不在已选集合$U$中权重最小的边，更新堆的复杂度$O(log(V))$，共遍历 $E$ 次，。

note：以下实现没有严格遵守「算法导论」的步骤，将遍历遇到的所有边都加入优先队列，堆中的元素会逐渐超过 V 个，导致插入和删除操作的时间复杂度提高至 $O(log(E))$。但如果要手动维护堆的话，需要调用更底层的 down 函数，增加了实现的复杂性。且$E < V^2，log(E) < 2log(V)$，应当只是带来常数上的改变而已。

```
// TODO：maintain heap manually
func minCostConnectPoints(points [][]int) int {
    size := len(points)
    graph := make([][]int, size)    
    for i := 0; i < size; i++ {
        graph[i] = make([]int, size)
    }    
    for i := 0; i < size; i++ {
        for j := i + 1; j < size; j++ {
            graph[i][j] = abs(points[i][0] - points[j][0]) + abs(points[i][1] - points[j][1])
            graph[j][i] = graph[i][j]
        }
    }
    return prim(graph)
}

func abs(num int) int {
    if num < 0 {
        return -num
    }
    return num
}

type edge struct {
    from, to, weight int
}
// implement priority queue
type priorityQueue []edge

func (q *priorityQueue) Push(x interface{}) {
    (*q) = append(*q, x.(edge))
}

func (q *priorityQueue) Pop() interface{} {
    pq := *q
    size := len(pq)
    x := pq[size - 1]
    *q = pq[:size - 1]
    return x
}

func (q priorityQueue) Len() int {
    return len(q)
}

func (q priorityQueue) Less(i, j int) bool {
    return q[i].weight < q[j].weight
}

func (q priorityQueue) Swap(i, j int) {
    q[i], q[j] = q[j], q[i]
}
// helper function for prim routine
func addEdges(v int, graph [][]int, seen map[int]bool, queue priorityQueue) priorityQueue {    
    for i, size := 0, len(graph); i < size; i++ {        
        if _, ok := seen[i]; graph[v][i] != 0 && !ok {
            heap.Push(&queue, edge{from:v, to: i, weight: graph[v][i]})
        }
    }
    return queue
}

func prim(graph [][]int) int {
    ans := 0
    cur, size := 0, len(graph)
    seen := map[int]bool{}
    seen[cur] = true
    var queue priorityQueue
    
    for i := 1; i < size; i++ {
        queue = addEdges(cur, graph, seen, queue)
        minEdge := heap.Pop(&queue).(edge)    // find minimal edge
        for seen[minEdge.to] {
            minEdge = heap.Pop(&queue).(edge)
        }
        seen[minEdge.to] = true
        cur = minEdge.to
        ans += minEdge.weight        
    }
    
    return ans
}
```
golang使用优先队列得自己先实现接口，所以代码有些冗长。

### Kruskal

并查集，所有边按照 weight 升序排序，从小到大的遍历过程中，如果边的两个端点不在同一个集合内则选择该边作为 MST 的边，且对两个节点进行 union。


时间复杂度：$O(Elog(E)+(V+E)\alpha(V)+V)$，主要开销来自对所有 edge 的排序，并查集的查询和合并开销$O((V+E)\alpha(V))$，并查集初始化$O(V)$ // TODO:并查集 阿克曼函数

```
func minCostConnectPoints(points [][]int) int {
    size := len(points)
    graph := make([][]int, size)
    for i := 0; i < size; i++ {
        graph[i] = make([]int, size)
    }
    
    for i := 0; i < size; i++ {
        for j := i + 1; j < size; j++ {
            graph[i][j] = abs(points[i][0] - points[j][0]) + abs(points[i][1] - points[j][1])
            graph[j][i] = graph[i][j]
        }
    }
    
    return kruskal(graph);
}

func abs(num int) int {
    if num < 0 {
        return -num
    }
    return num
}

type edge struct {
    from, to, weight int
}

func kruskal(graph [][]int) int {
    ans := 0
    size := len(graph)
    parent := make([]int, size)
    for i := 0; i < len(parent); i++ {
        parent[i] = i
    }
    edges := make([]edge, 0, size * size)
    
    for i := 0; i < size; i++ {
        for j := i + 1; j < size; j++ {
            if graph[i][j] != 0 {
                edges = append(edges, edge{from: i, to: j, weight: graph[i][j]})
            }
        }
    }
    
    sort.Slice(edges, func (i, j int) bool {
        return edges[i].weight < edges[j].weight
    })
    for i, count := 0, 0; i < len(edges) && count < size - 1; i++ {
        e := edges[i]
        if find(parent, e.from) != find(parent, e.to) {
            union(parent, e.from, e.to)
            ans += e.weight
            count++
        }
    }
    return ans
}

func find(parent []int, x int) int {
    for parent[x] != x {
        parent[x] = parent[parent[x]]
        x = parent[x]
    }
    return x
}

func union(parent []int, x, y int) {
    px, py := find(parent, x), find(parent, y)
    parent[px] = py
}
```

算法的具体步骤和插图如有机会再补，今天就先到这里。

### 总结

两个算法从渐进时间复杂度上其实都是相当的 $O(E×log(V))$[^1]，不过从算法运行过程上看 prim 主要开销是维护大小为 V 的堆，这与顶点的数量有关，所以稠密图使用prim更合适；kruskal 主要开销是对所有边进行排序，更适合稀疏图。

The End

---------


[^1]: [算法导论-23章最小生成树]()