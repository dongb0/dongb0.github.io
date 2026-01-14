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


abstract

提出一个启发式算法，将问题映射为 0-1 interger programming problem
// 是不是 BigSub 说的 ILP？ // 能保证最优解


1 Introduction
当时提供分布式、异构数据的访问可以通过：1）虚拟视图（适用于数据源变更较多的情况；2）数据仓库、物化视图（适用于数据源变化少，且要求对这些数据能够提供快速访问

1.1 Related Work

这项工作大概就是找 common subexpression； // 这句话是我自己写的吗？

一些更早的工作通过 operator tree（指逻辑/还是物理执行计划？）和 bottom-up 遍历的方式来找 common part；

也归属于 Multiple-Query Optimizing（MQO）领域内，但 MQO 与本文的工作有所不同：
- MQO 是要找出能用于多个查询、使得其中的 common expression （中间结果）能被最大化利用的最优执行计划；本文是要找出一组物化后能最小化执行代价的关系 // 或者说视图？
- MQO 似乎并不物化中间结果？似乎是这些中间结果驻留在内存中能被重复利用，比起串行执行查询，代价更低； 但本文会将这些 common expression 物化。
- MQO 的目标是提供最佳性能；本文的目标是综合考虑查询性能和维护代价；
- MQP输入是 workload（一组查询），输出是一组全局最优的查询计划；本文的workload是一组查询和每个查询的频率，以及base relation和对应的更新频率，输出是一组视图（需要物化的


本文算法能够面向 SPAJ query，对比了 Gup 97 的 AND、AND/OR graph，指出他们没有提供对算法质量（性能）的评估；

1.2 Contribution

- 提出 MVPP framework 用来描述问题，似乎算法也能用于分布式数仓？
- 提供 cost model,分析查询性能和视图维护代价
- 介绍了生成 MVPP 的算法和视图选择算法，生成MVPP（候选）可以采用 “优先速度”或者“优先性能”（后者采用 0-1 integer programming problem 映射来找最优

2 Issue of materialized View Design

2.1 Motivating Application
一个分析供需关系的warehouse例子

2.2 Example MVPPs
对于2.1给出的简单例子，可以（肉眼）找到其中一些公共的expression，
当然我们可以有三种物化的方案：
1）直接物化整个 query，维护开销很大（想想对于如今更多查询的例子就不适用了
2）全都不物化，性能很差
3）物化一部分节点（选择其中能带来更高性能和相对较少维护开销的

3 Specifications and Cost Analysis for Materialized View Design

MVPP 是能够表示查询执行（处理）策略的 DAG；叶节点表示基础表，根节点表示query（一组查询根据生成的查询计划，可能有许多不同的 MVPP 结构；

3.1 Specification
// 这里是对 MVPP 的定义，公式不方便敲就不重复了，记点理解；

- 每个关系代数符号（运算符）、每个基础表、每个查询，都对应一个顶点
- 用T(v) 表示由v生成的顶点，可以是基础表、中间结果、或最终查询结果  
...
- 如果中间节点 T(u) 需要进一步处理，则引入一条边（弧） T(u)-->T(v)
- 定义 descendant node S(v) ：有边指向 v 的
- 定义 ancestor node D(V) ： v指向的
- 定义查询开销 Cq(v) (通过访问 T(v) 的路径)；维护开销 Cm(v)，是 S(v) descendant 的闭包 和基础表 R 的交集 // 为什么是与 R 的交集？

随后可以用 MVPP 来描述视图选择问题：
选择一组顶点V，使得T(v)物化后查询的处理开销和视图的维护开销总和最小

还可以用上述定义描述 MVPP 的生成过程：
找到所有的顶点 pair \<u, v>，使得他们的 S'() 闭包和T()相等，则T(u) T(v) 就是 common subexpression,可以被合并

3.2 Cost Analysis
3.2.1 Cost of an MVPP
给出最终的目标函数 C_total 定义，大体来说就是  查询频率 * 每个查询使用某个视图v的开销 + 维护频率 * 每个视图v的维护开销， 求和


3.2.2 Cost of Shared Views

定义度量 Ecost(v),是查询开销与共享视图v的查询数量的比值，（也就是选出的 v 能被越多查询共享，认为其收益越高）//而不是实际节省的开销？

4 Algorithm for Materialized View Design

4.1 Algorithm for Selecting view to be materialized
对于一个包含 N 个顶点的 MVPP 来说（除去leaf），我们有 2 ^ n 种选择方案（如果要遍历的话），但是可以使用一些启发式方法来缩小搜索空间。

贪心的原理：如果物化一个节点带来的 saving 大于维护他的开销，那么就可以物化该节点。
// 或者说不是贪心？单纯启发式思想，而且不考虑存储限制，维护限制也只是要求其小于带来的收益

4.2 Algorithm for multiple MVPPs design


4.2.1 Transformation
考虑Join操作的成本最大，因此这里将 SPA 运算都 pull up；另外在生成 MVPP 时将这些运算 push down（优先考虑，能先把数据筛选的都筛选掉）

根据一些 rewrting 规则就可以完成

- pushing down select operation：如果有查询share join op，且对与 join 的 select 条件不同，那么可以对 base relation 预先使用 所有 select 的析取 进行筛选；（这好像是比较直白的下推操作）；举个例子，比如 
select * from t1 join t2 on t1.col1=t2.col2 where t1.col2 > 1000;  
select * from t1 join t2 on t1.col1=t2.col2 where t2.col1 < 4096;
则这个 join op 的筛选条件是（t1.col2 > 1000 and t2.col1 < 4096）// 注意是用 AND！
- ～ project op：对于投影（也就是 select 部分的列），我们取这些列的合取
- ～ aggregation with identical groupby：新的aggregate 需要包含所有原本的aggregate function //有点不太理解新的指什么
- ～ aggregation with different groupby： // 什么勾八东西看不懂

4.2.2 A feasible Solution
如果希望能快速选出一个还行的solution，我们可以使用每个查询的最优执行计划（也就是优化器推荐出来的那个），按照查询频率 乘上 查询开销 来进行排序。
然后尝试将 2～n 的查询计划合并到第1个中，形成一个 MVPP；
随后第二轮迭代尝试将第1个和3～n的计划合并到第2个中，也形成一个 MVPP；

最后我们将会得到 n 个MVPP；然后使用 4.1 中的视图选择算法，对比每个 MVPP 的开销，从中选择出总开销最低的那个。

这里 生成 MVPP 的算法没有例子看不懂，不要紧，另一篇论文里有例子；

4.2.3 A 0-1 integer programming solution

// 

5 Conclusion

...应用的场景只是由几条查询语句构成的小规模数据集


A Framework for Designing Materialized Views in
Data Warehousing Environment

算法还有一小部分没明白


看不懂，垃圾
垃圾
垃圾
垃圾垃圾
垃圾
垃圾
