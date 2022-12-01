---
layout: post
title: "[]"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - paper
---

// 看实验结果似乎是把 mv 转化为 树？（有些特殊情况可以转化为平衡树？，
// （不过一般情况还是有向无环图，所以其实两个算法，只是简单的贪心？



abstract

视图选择在维护数据仓库的高效运作中是非常重要的。
需要选择合适的视图，以便最小化总查询时间、维护视图的开销，同时还能满足一定的资源限制条件（如视图更新的时间、存储空间和物化视图所需要的时间等

本文提出算法，（尽量）满足上述目标；
首先提出一种近似贪心算法，在限制维护成本的场景下，解决 OR 视图 的视图选择问题；
证明 query benefit 在最优解的 63% 以内；//哪来的证明？

第二，对于 AND-OR view graph 设计了一种 A* 启发算法，（提供最优解？（deliver an optimal solution?

且实现了第一种贪心算法，进行了实验；


1 Introduction

这篇文章主要是面向数据仓库进行视图选择的，数据仓库通过 integrator 从数据源处获取数据，并维护数据仓库中的视图

文献
Selection of views to materialize in a data warehouse
定义了（数据仓库中的）视图选择问题：给出一组查询请求，在给定资源限制下，选择一组视图进行物化，使得总查询响应事件和维护视图的时间都最小化

那篇文献提出一种接近最优的多项式时间贪心算法（但只能应用在仅考虑空间占用的情况下

本文提出的算法将考虑视图维护时间的限制。这比考虑空间限制要复杂很多，因为视图的维护开销需要考虑其他视图带来的影响（ // 可以看看具体如何影响的

对于 OR view graph（//到底什么是 OR view graph：从另一个视图获取的视图（它是被获取得到的还是用来获取其他视图的？
也提出一种接近最优的解法；

sec 2 motivation
sec 3 definitions
sec 4 define maintenance view-selection problem
sec 5 greedy algorithm for OR view graph
sec 6 A* algor for AND-OR view graph
sec 7 experiment
sec 8 conclude

1.1 Related Work
列举的所有视图选择算法，都仅仅考虑了空间限制，

另外的考虑视图维护代价的算法，大多是通过穷举搜索，或者不能保证 solution 的质量和性能

Ours is the rst article to address the problem of selecting views to materi-
alize in a data warehouse under the constraint of a given amount of total view
maintenance time.

2 Motivation and Contributions
之前的为视图选择设计的多项式时间渐进算法都只能在 限制硬盘空间 的场景下，
因而在现实场景中应用有限（磁盘比较便宜，
现实中更多限制来自于维护物化视图的开销（主要是数据更新时维护视图的时间成本太高，我们才无法将每一个视图都物化

因此本文提出了一种能在视图维护时间限制下，进行视图选择的算法。

在空间限制或者维护代价限制的约束下进行视图选择都是 NP-Hard 问题。

在 space constraint 情况下的视图选择，某个视图的 query benefit 并不会因为其他视图的物化而变化，因此未选择视图的 benefit per unit space 总是单调减少的。（为何是单调递减的？

在维护开销限制的场景下，一个视图的维护开销可能随着其他视图的物化而减少；因此同时使用两个视图的维护开销，可能少于这两个视图都单独维护时的开销总和；这导致 query benefit per unit maintenance cost 不是单调的，可能出现同时使用两个视图时的增益大于分别使用这两个视图得到的增益。因此之前针对 space constraint 的贪心算法可能会得到比较糟糕的 solution。

Contributions
1）maintenance cost 的算法
2）对 OR view graph 提出一种贪心策略，并证明该算法能得到近似最优解
3）对 OLAP warehouses （候选视图能构成 OR boolean lattice），提出一种 A* 启发算法

3 Preliminaries

定义2 （Expression AND-DAG）
有向无环图 V，中没有出边的基础表（或mv）被称作 sinks ，没有入边的节点被称作 source 

如果 node/view u 有出边指向节点 v1,v2,...,vk, 那么 u 能通过**这些**节点计算得到（//不需要全部是吧？），这样的依赖称为 AND 弧

这样的 AND 弧通常有一个operator 和 查询代价 与之关联，该代价表示通过 v1,v2,...,vk 计算得到 u 的开销。

定义3 （Expression ANDOR-DAG）
ANDOR-DAG 是基础表作为 sinks、视图V 作为 source 的有向无环图。每个 non-sink 节点  v 与一个或者多个 AND 弧关联

定义4 （AND-OR View Graph）
对拥有基础表作为 sink 的有向无环图 G，如果每个 Vi 都存在一个子图 Gi 构成 Vi 的 ANDOR-DAG，那么 G 被称为一个 AND-OR view graph；

Given a set of queries Q1 ; Q2; : : :; Qk to be supported at a warehouse, \[Gup97]
shows how to construct an AND-OR view graph for the queries.

// 可能最好得去看看提到的这篇文献；

定义5 （Ealuation Cost）
AND-OR view graph 中的 AND-DAG H 的 总cost 是与之关联的所有 AND 弧的 cost 之和，加上与 H 关联的 reading cost


定义6 （OR View Graphs）

AND-OR 图的特殊情况：AND 弧只与一条边关联。

4 The Maintenance-Cost View-Selection Problem

问题定义用语言概括来说，是给定一个 AND-OR view graph G 和 一个维护时间 S，mainttenance-cost view-selection 问题定义为选择 G 的一个子图 M (M 是一组视图集合)来最小化查询响应事件，同时维护视图集合 M 的开销不超过 S。

这里用 Q(v, N) 表示使用视图集合 M 响应一个查询 v 的时间开销； UC(v, M) 表示在 M 集合中维护视图 v 的开销。

// 两条公式 定义这个问题，比较好理解就不敲公式了，记住符号 U(M) 表示 M 的总维护时间；

计算 Q(v, M)
相当于从 AND-OR view graph 中找到权重最小的 AND-DAG Hv，且 Hv 属于集合 （M 并 G 中的 sinks）。这里假设 L 总是可达的。

OR view graph 中计算 UC(v,N)
UC函数取决于使用的 维护代价 模型

但所对于 OR view graph 来说，UC的计算定义为 从 v 到某个节点 u 的最小维护路径（即路径上维护开销之和最小）

5 Innverted-Tree Greedy Algorithm

提出 Inverted-Tree 贪心算法，能在 OR view graph 中提供近似最优解；

---------------
最早使用贪心算法来解 视图选择问题的是 \[HRU 96]，限制条件是磁盘使用空间
\[Gup 97]扩展到 AND-OR view graph 的一些特殊场景，限制条件同样是磁盘使用空间
---------------
// 是否要考虑看一下这些文献？

Most Beneficial View
在视图集合 M 中增加视图 C，C 带来的增益是 使用 M 时的开销，减去 M 并 C 后得到的开销；

而维护视图 C 的开销 effective maintenance cost 计算法方式是 U(M并C)-U(M)

由这两项定义，我们可以定义算法需要的 most beneficial view
// 但每个阶段都选择 most beneficial view 可能会得到糟糕的 solution？

// Q：为什么这里给出的维护开销，只有跟基础表关联的两个视图才有？如果这些视图都是要物化的，那么每个视图应该都对应有一个维护开销啊？


Definition 7.（Inverted Tree Set）

inverse graph of Tr is a tree；

提出 Inverted Tree 的动机是：我们观察到 OR view graph 中的任何集合 O，都可以被分割为 inverted tree set，使 O 对于已经物化的视图 M 的维护开销，大于 inverted tree set 对于 M 的维护开销；

由此提出了对应的贪心算法，主要考虑视图集合中所有的 inverted tree set，选择其中 每单位维护开销得到增益最高的（most query-benefit per unit effective maintenance）

Definition 8 (update graph)
OR view graph 中的子集 O，O 中每个节点的更新开销，小于等于 任意节点 w
UC(u; {v}) <= UC(u; {w}) 对所有w属于O都成立
// 应该是 这些 UC 是最小的一批？ 那岂不是所有 UC 都是同一个最小值？ 或者说他们都能走同一个路径？


Lemma 1 证明该贪心思想的合理性
OR view graph 中的一组视图集合 O 能够被分割为若干 inverted tree set O1，O2,...，Om，使得 这些 Oi 的维护开销总和小于 对整个视图集合 O 的维护总和；

Proof：
由定义，

// 看不懂

Theorem 1（证明算法能得到近似最优解）
给定一个 OR view graph G 和 总维护时间限制 S，该贪心算法能够返回 solution M 使得 U(M) <= S2 且 M 的 benefit 最少为最优解的 (1-1/e) = 63%；

如何证明？

Theorem 2 
大意是近似最优解的增益大于最优解的 0.5 ？
也没看到证明啊

Time Complexity

结论，最坏情况指数时间复杂度，// 为什么是指数复杂度，解释没看懂
复杂度是 inverted tree set 的数量，
因为节点 v 的每个祖先都能构成一颗 inverted tree，

如果 OR view graph 构成平衡二叉树，那么算法的复杂度为O(n^2)，n是节点数量；

6 A* Heuristic
把节点排序（inverse topological order）之后，构成一颗二叉树，用于搜索最优解；

// 思路可能是把 maintenance-cost 的限制，转变为 二叉树搜索上的障碍；但是每次都往启发函数的方向走，如何能保证一定是最优解？总的查询延迟如何保证最小？

// 算法过程我咋看不懂啊？又怎么保证最优解？怎么考虑维护开销？

// 要不先看看 A* 算法？
启发式, f(x) = g(x) + h(x), h(x) 启发函数，g(x) 是距离起点的距离，f(x)是综合优先级，如果 h(x) = 0，则变为 Dijktra 算法；

这里的 h(x) 计算方式为，

s(v) 是 v 的最小维护开销
最小查询开销 p(v)
p(v)/s(v) 

最坏情况同样是指数时间复杂度，

且 A* 的空间开销 比贪心 大很多 （好像是指数级别？
