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

Q1：benefit-per-storage ratio 大概是怎么计算的？

Q2：图里的 relative workload cost 含义？标点的地方是意味着算法找到 best solution 了吗，如何验证它是 best 的呢？

Q3：如何对这些算法的具体步骤有更详细直观的认识？看原论文还是看代码？// 难道都看？

Q4：优化器如何估计查询执行的代价/开销？文章说精确计算索引带来 benefit 可以通过创建索引并实际执行的方法，但是开销大，那么实际通过什么方法来估计呢？是通过数据规模来估算使用索引后减少的IO次数吗？

索引选择要解决的问题：
在没有DBA指定索引的情况下，数据库通过索引选择算法，针对某些工作负载，自动创建索引，提高查询的执行效率；

abstract

主要挑战：1)如何确定一个index的workload cost 2)批量查询和schema导致（index selection）的多种组合，（如何选择？）

本文从solution quality，运行时间，多列支持，solution粒度，复杂度等多种维度介绍并比较八种 index selection 算法，

1 INTRODUCTION

数据库通常使用二级索引加速查询，但对于某个workload如何选择最优的index set来查询，问题就比较复杂：

1)solution space。可能的 index 组合数量非常巨大（可创建的 index 数量随着 attribute 的增加而增多），同时还需要考虑 index 类型，如Btree，hash map，bit map。
2)index之间的影响。“the benefit of an index may depend on the existence of other indexes” ？
3)量化 index 对于某个 workload 的代价也是一项有挑战性的任务。

早期的相关研究主要关注寻找优化的索引配置。但在 calculation time，solution quality，optimization criteria 和 cost estimation 上都没有针对性/没有统一？（是否这么理解？

本文在更多维度上对不同的 index selection 算法进行了对比，（以及在三种 workload 上进行了算法性能的评估）

Contributions：
提出对8种 index selection 算法的评估工具？

分别是 Drop Heuristic，Autodmin's Index Selection, DB2 Advisor, Relaxation, CoPhy, Dexter, Extend, DTA Anytime

- explanation & analysis
- evaluation for 3 benchmarks, TPC-H, TPC-DS, Join Order
- diff parameters infuence
- 对不同用户需求的推荐
- open-source evaluation platform

-------------

sec 2 formalize index selection problem
sec 3 describe & analyze these algo
sec 4 methodology
sec 5 evaluation platform
sec 6 experimental results
sec 7 findings
sec 8 conclude

------------

2 Index Selection Problem

2.1 Formalization
trade-off between performance and storage consumption。

但 index 也并不总能加速查询的执行，比如 insert/update 需要更新更多 index 也会导致执行的速度变慢。

index selection 的目的是找到一组 indexes 在执行某个 workload 时能获得最佳性能（考虑存储空间限制、创建 index 种类、运行时间等约束的情况下）；

workload 定义：
工作负载由一组查询 Q 构成，每个查询 j 包含 {1,...,N} 个 attribute。

整个工作负载的开销定义为所有查询 j 开销的总和。
每个cj的开销取决于选择的索引。因此每个查询出现的频率也需要在进行索引选择时纳入考虑。

Index 定义：
一个拥有 W 个 attribute 的索引 w 可以表示为 w={i1,...,iw}，W 称为索引的宽度。索引的 size 则是对应的存储开销。

一个索引只属于一个 table。

Index Configuration 定义：
索引配置？ k 是一组索引的集合。可以用 index configuration 来表示查询和工作负载的的开销。// 公式

Potential Index 定义：
可以在工作负载中产生包含 W 个 attribute 的索引 p 

Index Candidates 定义：
顾名思义。通常 candidate 数量多，算法无法对所有候选索引进行评估，许多算法会倾向于选择与查询相关的索引，比如索引中的列至少在某个查询中全部出现。

Index Interaction 定义：
一个索引的 benefit 可能被其他索引影响\[45] // 有空大概翻一翻
因此选择索引时需要综合考虑

Index Selection 定义：
Index Selection S 是从 candicate 集合 I 中选出的索引集合。

Hypothetical Index 定义：
假想索引（虚拟索引）并非实际存在的数据结构，它们的用处是让优化器模拟评估查询计划的开销。

2.2 Determining Query Costs

最精确的量化某个 index 代价的方法是，实际创建该 index 然后运行 query，以 query 的实际运行时间作为度量。

但是这种方法开销很大。

故而大多数 index selection 算法并不会以这种方法来衡量，而是使用 optimizer 和代价模型来评估使用某个索引的开销是多少。某些数据库甚至支持 hypothetical indexes，并不实际创建索引，而只是在 cost model 中用来估计执行计划的 cost。
这种方式在文中被称为 “what-if calls”，在大规模的 workload 中，what-if calls 可能会变成性能瓶颈。（sec 6.5）

可以通过 1）减少需要评估的 configuration 2）reducing cost from simpler config ? 来减少这些开销。
> 不太明白这里说的 configuration 是什么，有论文通过 cache 的方式减少了 what-if calls 的数量，作者声称比常规方法快3个数量级，而精度并无损失。\[10]

optimizer-based cost estimation 可能因为不准确的cost model \[50] & cardinality misestimation 而导致评估错误。\[28]
算法的质量一定程度依赖于 cost estimation 的准确性。
啊？怎么就神经网络和机器学习分类算法了？ 预测哪个执行计划更高效。

有提到 \[30] 中基于神经网络的代价模型，似乎能得到更准确的估计结果？
\[15] 中通过机器学习的分类方法，//不知道到底干了什么。帮助预测执行计划是否准确？

3 Index Selection Algorithm

3.1 Drop
最早的索引选择算法之一，运作过程大约是一开始的候选集合中包含所有的 single-column index，然后每轮 drop phase 尝试从候选集合中删除一个索引，再对整个集合的代价进行评估，如果某个索引的删除可以得到更低的开销，那么该索引永久从集合中移除，重复该过程直到开销不再降低。

代价的度量：characteristics of the data？（原论文）// 不理解
// 这里是说，某个 configuration 的代价，是由 data 的特征决定，而不是通过 optimizer 估算得到； 1971年的算法，当时似乎优化器还不能估算代价？ // 这一点待验证

参数：可配置选择索引的数量。

3.2 AutoAdmin
用于SQL-Server。 multi-column index，iterative

迭代分为两步:
1 通过遍历每个查询，来确定每个查询对应的候选集合 Sj，j=1...Q，合并所有候选集合 Sj 作为第二步的输入； //也就是生成候选索引的方式是暴力枚举？
2 第二步与第一步区别只在于考虑查询（的范围/数量）和候选索引（数量）

枚举整个候选集合中，m 个元素的子集；如果 n = m 则表示枚举所有子集，这能确保找到最优解，但是时间复杂度非常高；如果设置 m = 0 则退化为贪心

主要就是枚举候选索引的集合 with m element？然后贪心寻找 n > m 的索引，如果 m=n 则枚举评估所有索引子集，来确保得到最优解；如果 m = 0 则用村贪心酸来降低运行时间？

使用单列索引用于第一次迭代，然后创建双列索引，用于评估和进行下一次迭代。

为了得到 multi-column 的索引，作者提出两种策略：选择更多索引（更优结果）、更少索引（更短运行时间）

参数：最大索引数量，索引宽度，枚举索引数量

3.3 DTA
SQL Server
对AutoAdmin的优化，同样根据每个查询获取候选索引，然后根据整个 workload 来贪心选择 index configuration。

优化：1）不再增加 index 宽度。2）对 candidate 选择后，合并剩余的 candidate 和 config 来进一步筛选对于 multi query 有提升的 candidate。 3）通过 seed configuration 来考虑 Index interactions。// 筛掉一些 次优解，避免完全枚举所有组合 4）tuning time 可以根据 solution 的要求来选择？ 5）算法可以用于 tuning index、mv 和 partitioning criteria

// 是否整体来说还是一种枚举的算法；

参数：索引的最大宽度，tuning time 的限制

3.4 DB2Advis
显然用于 IBM DB2
optimizer what-if 

算法分为三步：
1 确定候选索引，对工作负载中的查询 1～Q，对 等式/范围谓词/order list 中的列创建 hypothetical indexs；同时为了不遗漏 candidate，所有潜在索引都将作为 hypothetical index 加入到集合中（除非索引数量达到阈值）
随后优化器挑选对于查询 j 的最优计划，被计划使用到的索引将加入候选索引集合 I 用于下一步
2 所有候选索引 I 按照 benefit-per-space 的比值顺序进行降序排序。
如果两个索引 w1 和 w2 拥有相同的列集合（if w1 has a higher ratio,
and its leading columns are equal to w2 ’s column permutation），且 w1 的比值更高，那么算法会将 w1 和 w2 合并。然后移除 w2，对剩余的索引重新排序。
最后将所有索引加入最终的 index configuration S 中（直到超出配置的内存大小）。
3 将 Index Configuration 中的索引随机 varied？如：将 S 中的索引集合与（因内存限制）不在 S 中的索引交换，如果能得到更低的开销那么就将其作为新的 Solution。

参数

3.5 Relaxation
提出一种从（较大的）最优索引集合中，通过重复转换（transform）来降低存储消耗，得到新索引集合的方法。

首先通过每个 query 获取最优索引集合，（...）通过一些优化避免暴力枚举
第二步，将所有查询的最有索引集合合并得到整个工作负载的索引集合，随后重复进行 relaxed操作，来在减少索引集合的存储开销的同时，尽量不损失性能；（how？）
使用了五种索引转换方法：1）合并两个索引 2）分裂两个索引？3）将索引的 attribute 从末尾开始移除？ 4）将索引变为聚簇索引 5）删除索引

参数：

3.6 LP-based Approaches
Linear Programming 线性规划
可以将索引选择算法转化为线性规划问题。（似乎是 off-the-shelf?

GUFLP
限制每个查询只能使用1个索引

Cophy
考虑了每个查询可以使用多个索引，优化器可能选择多个查询计划。
算法核心思想：
设索引组合 K，将某个候选索引子集 k 属于 I，称为 option，应用到 query j 上，对应的开销为 Cj(k)，k 属于 K 并 {0}

LP 问题中，二元变量 Zjk 用于表示索引 option k 被应用于 查询 j 上。二元变量 Xi 表示 索引 i 是否被选择？LP 的约束保证每个查询 j 使用唯一索引 option ，且使用的索引 i 不超过存储限制 A。

LP问题的复杂度是由变量和约束的数量决定的，LP 公式需要所有的代价系数 Cj(k)，且无法扩展；
因此1）无法用于大规模的问题，2）可能导致次优解，受限于候选集合规模


3.7 Deter
postgreSQL的索引选择工具。
基于 hypothetical index，分为两个阶段；

首先处理查询，从 plan cache 得到查询的运行时间，相同模板的查询分到一组
随后，1）通过 EXPLAIN 命令获取初始的查询开销 2）创建 hypothetical index 3）从查询优化器中获取开销估计和查询计划，创建的虚拟索引如果被使用，则将其加入索引候选 4）选择所有候选索引中，比起步骤1的初始开销优化最多的那个，作为 solution。

index 不能超过两列

参数：

3.8 Extend
iterative selection for multi-column

贪心，每步根据最高的 benefit-per-storage 比率扩展索引或添加索引。（how？
从空集开始，将 cost reduction per storage 比率最高的索引加入集合 S，随后有两种选择：
1）加入新的单 attribute 索引（与S无交集）
2）将 S 中某个索引扩展（添加 attribute
（同样根据 best ratio 选择

内存占用达到阈值，算法中止。

参数：

3.9 High-level Comparison

选择 index 时主要考虑 benefit per storage consumption；
算法停止条件也有所区别，
主要还是看论文里的表格。

3.10 Machine Learning-based Approaches
有若干基于机器学习的索引选择方法，不列举
主要基于 TPC-C workload，忽略 index interaction；

Limitaton：

3.11 商业索引选择工具

选择的算法也许并没有全面反映这些工具的性能；
DAT
DB2 Advisor


4 METHODOLOGY
主要谈论 evaluation setup，大致理解为评估的标准和步骤

4.1 Workloads
使用三种 workload 作为 benchmark，（不重复

4.2 Determining Query Cost
使用 虚拟索引 来评估开销；这必然与实际的结果存在一些误差，但是综合速度、准确率和可用性来说是比较好的选择；

因此选择使用PG with HypoPG 扩展（支持创建、删除虚拟索引，以及能够估计虚拟索引的size）来进行评估开销；

因此可以通过 EXPLAIN 命令来确定使用哪些虚拟索引；

发现使用虚拟索引的估计能够比较好的贴近使用真实索引的开销。

4.3 Constraints and Optimization Goal
Optimization Goal
虽然这些算法可以针对ratio of benefit and size进行优化，但是论文中对比的还是原始实现。

Limit
Drop，Autodmin 限制的是索引数量，Dexter 。。。能通过各自的限制手段来控制存储占用，
不过更多算法采用的是限制存储空间的占用

Runtime
只有DAT支持控制算法的运行时间

但一般来说运行时间可以通过限制候选索引的数量来控制 / 或者 evaluated configuration


5 Evaluation Platform
// 是否可略？

提供源码和平台让读者能够重现论文中的结果。

5.1
平台 python3 实现，自动化setup（生成和导入数据，生成查询，调节参数，记录结果）

在平台中实现了提到的几乎全部算法。

5.2 What-if Call Optimization
1)减少不必要的优化器调用，2）优化器维护执行代价的缓存，

6 Ealuation
6.1 Experimental Setup

6.2 TPC-H
1）大多数算法不会用完10GB的空间配额；
2）使用index数量作为停止条件与根据使用空间配额作为停止条件的算法在表现上也有所不同

--------
6.3 TPC-DS

6.4 Join Order Benchmark
--------
基本都是对比介绍图表中哪些算法在哪种工作负载下表现如何

6.5 Cost Breakdown and Cost Requests

算法的大部分时间消耗主要来自于向优化器请求评估某个query（在某种索引组合下）的执行时间；如果查询条件、索引组合和存储数据没有发生改变，优化器评估的开销应该是相同的，所以大部分算法都会缓存这些评估信息；

在实验中，大部分算法的 what-if calls 缓存率都挺高，但尽管做了缓存，这些请求的时间开销还是占了算法运行的大部分时间；

7 Learning

General Insights
1）算法在不同工作负载和限制下，表现不同（根本废话）2）算法选择应根据用户对不同条件的要求 3）基于查询（query-based）的算法，会评估所有潜在索引；基于组合的算法（combination-based）倾向比较较小的索引集合，相对来说后者能更多的评估索引之间交互的影响；但前者更快 4）what-if calls 的开销并不固定，取决于查询和索引组合；
5）solution的粒度？
6）停止条件很关键（这种算什么finding？
7）根据 benefit per space 来对候选索引排序比直接根据 benefit 排序更高效
8）Reduction-based 方法在可用空间配额更大的场景下运行更快，但在实际应用场景他们会慢



8


