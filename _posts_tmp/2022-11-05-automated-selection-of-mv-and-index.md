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

// 主要贡献
// 1)将 mv 和 index 看作类似结构，进行 Joint Enumeration，通过 optimizer？（待定 得到二者统一的增益度量（但是文中怎么只提到  mv 的度量？
// 2）提出了一种算法（简单的启发式）来缩小搜索空间，但为了不遗漏最优解，又添加了一个 merged 算法，通过合并 mv 来扩大候选集合；（这一步还是会根据 mv 的数量进行一次筛选防止过度增加搜索空间
// 最后还是应用贪心算法来进行搜索； 比如 greedy(m,k)

// 局限：同样是贪心，同样是静态；

abstract 

选择 mv 和索引的 solution，在 SQL Server 2000 上实现；

1 Introduction

This richness of structure of
materialized views makes the problem of selecting
materialized views significantly more complex than that
of index selection.


需要 innovative techniques 来解决 large search space 的问题；

批判了一番之前的工作，只会在整个 候选集合 中 search，而没有考虑要简化候选集合；

1）提出新架构和算法来解决上述问题
2）提出一种能够生成 smaller 候选mv集合的方法，大大缩短了搜索 mv 的用时


1）提出一种方法来得到 interesting set of tables，mv 只需要在这些 table 上进行（因而缩小了搜索空间；
2）提出一种称为 view merging 的方法，来辨识那些对 query 没有帮助的 mv；（也进一步缩小搜索范围

2 Architecture for Index and Materialized View Selection

架构图大概描述来说，是 workload 经过句法分析，得到候选的 index 和 mv，与 Estimation 模块评估交互之后，得到 Configuration，然后从中选取得到最后的 mv/index 集合。

这里的 workload 同样是静态的一整个，文章建议可以从日志中得到，或者使用制定的 benchmark；


得到 workload 之后，第一步工作是先进行句法分析，来得到候选的 index 和 mv；
比如！对于查询：
  select sum(sales) from Sales_Data where city = 'seatle';

以下的查询都是句法相关的（syntatically relevant）：
v1 select sum(sales) from Sales_Data where city = 'seatle'; // 完美mv(对于这个查询来说)
v2 select city, sum(sales) from Sales_Data group by city; // 需要一些筛选
v3 select city, product, sum(sales) from Sales_Data group by city, product;

// configuration 到底指的是要被搜索的候选集合?还是搜索得到的 physical design？
// 看起来像是前者

同样贪心奥，采用 \[4] 中的贪心算法
大概思想是算法最终会得到 k 个 index/mv，而一开始先通过穷举的方式选择 m 个最优的 
随后使用贪心来获取剩下的 k - m 个
// 但是贪心的方法是什么？

文中算法的 configuration enumeration 范围是 indexes 和 mv 的总和（二者搜索空间的并集

本文在 SQL Server 中实现了能根据 查询 Q 和 configuration C（哪怕是虚拟的 C）来计算 Q 的开销 的 模块

文献 \[5] 是关于 what-if structure 的

3 Related Work

...略去一部分



4 Candidate Materialized View Selection 

// 这里大概说了怎么筛选，整体上也确实是 mv 的生成与筛选，但比较关键的一步：如何从 sql 到 mv（自动化地），这里没有描述（用的伪代码

要搜索 workload 的所有相关mv 是不可能的，数量太多了：
如果查询有 m 个选择条件，涉及 table subset T ，那么包含这 m 个选择条件任意子集的 mv 都属于 syntatically relevant；

因此需要一些策略来缩小搜索空间；

首先要明确的一点是：我们无法对每个 qeury 都选出一个能够正好匹配的 mv；
比如有些查询中会出现子查询，但是 mv 中不会包括子查询；

同时，在空间限制的场景下，忽略查询之间的共性（commonality）也可能会得到次优解；


第二点需要关注的是：一部分 table 即使我们提供了相应的 mv，对整个 workload 带来的增益也很少；比如一部分很少被访问的 table；
即使是经常被访问或者出现在 expensive query 中的 table subset，其中每个 subset 被访问的概率也并不一致；
如，TPC-H 数据集中，一章表包含 lineitem, orders, nation,region，其中nation和region对应的行数为数十行，而 orders和 lineitem则有数百万行，显然预计算 orders和 lineitem的收益会更高；

基于以上考虑，我们将 候选mv 的选择任务 划分为 三个步骤：
1）对于 large space 的workload，我们需要按照 sec4.1 的方式选出一部分 interesting table-subsets
2）基于这些 interesting table-subsets，我们为 workload 中的每个 query 设计出一组 mv，并从中选出 best configuration（sec 4.2
3）从这些 config 开始，我们生成 “merged” mv，sec 4.3，并上 步骤2）中的 mv 来组成完整的 configuration enumeration；

4.1 Finding Intereting Table-Subsets

我们想找到那些 物化后 能够显著降低（比如超过某个阈值）开销的 table-subset；因此第一步是先定义 metric 度量

// 这里 table-subset 是指 tbale 中某几列的 attribute

文中说，定义 metric TS-Cost(T) 为 workload 访问某个 table-subset T 的开销总和，但这样存在弊端，比如 上文的 TPC-H workload 同时访问所有字段，那么 T1={lineitem, orders} 和 T2={nation, region} 的 cost 就相同，但显然物化前者的收益更高；

因此文中提出的度量为 

TS-Weight(T) =  cost Q * (T 的 size) / Q 访问 table 的size

因此我们通过以下两步来获取 interesting table-subset：
a）通过 TS-Cost 来筛掉未达到阈值的 table subset
b）将剩余的 subset 计算 TS-Weight 再筛选一次；
从而减少 搜索空间

设置 C 的阈值越低，则筛出来的 subset 越多，搜索空间越大；实验结果表明设置 C=10% of total workload ，得到 solution 质量并不会明显下降，但是运算速度会明显加快；

该算法用自然语言描述来说，基本就是上述 a),b)两步，从 size 为 1 的 T 一直搜索到 MAX-TABLES（这是 workload 中单个 query 引用 table 的最大数目），对其中的 table subset 计算 cost，超过阈值的加入结果集；


4.2 Exploiting the Query Optimizer to Prune Syntactically Relevant Materialized Views

通过 4.1 中算法进行筛选我们减少了最后 枚举算法所需要搜索的 数量，但这不够；本节通过 optimizer 进一步减少那些无用的 mv；

基于这样的思想：如果一个 mv 不能构成对单个 query 的最佳 solution，那么它大概率也不会参与构成整个 workload 的最佳 solution； // 典型贪心思想

// 有证明吗？有没有反例？比如一个作为 best solution 的 mv 能给两个查询提供增益，但这两个查询各自有一个 更小的 mv 单独收益比它要高？ 感觉似乎会存在反例，是不是要想一想？

这里反正就先设定了一个函数接口 Find-Best-Configuration(Q,S) ，从集合 S 中返回 Q 的 best configuration；里面的具体实现可以套别的算法，比如 sec 2 中的 Greedy(m, k)；


接下来要考虑如何生成每个查询 Qi 需要搜索的 mv 集合；
但需要明确的一点是：那些正好包含 Qi 访问的 table 的那些 table-subset，也许并不适合，如可能在 sec 4.1 中被当作不感兴趣而没被选上；另外就是定义视图的方式跟查询中包含的 table 并不一致，如子查询不会出现在视图中；

那么要怎么做呢？

对每个 interesting table-subset T,
1）选择 包含 join 和 selection 的 pure-join mv
2）如果 Qi 有 grouping，那么选择 包含 group by column 和 aggregation 的 mv；

---------
这一段有些没看完，没仔细看



4.3 View Merging
然而，假如我们在 sec 4.2 筛选过后的configuration 中进行枚举搜索，在空间限制的约束下依然可能得到次优解；

所以需要我们考虑某些也许不对单个查询最优、但是会对多个查询有增益的 mv； // 我就说可能存在反例嘛

但是如果我们要通过考虑多个 query 的方式来添加 relevant mv 那么 搜索空间还是会爆炸；

所以我们这里采用的方法是：sec 4.2 返回的 mv集合 M 会包含基于 cost 选择的 mv ，因此大概率会被 optimizer 使用；因此我们可以将 M 作为 start point，来合并生成一些额外的 merged mv，以此来利用 M 中视图的共性；

这些生成的视图，并上 M 本身就是我们最终算法需要搜索的 candidate；

pair-wise merge；我们需要考虑：1）何时、如何判断两个 mv 需要合并 2）搜索所有可能的 merged mv；

详情见剩余两小节。

4.3.1 Merging a Pair of Views

将两个 parent view 合并为  merged view；要满足：
1） parent view 能响应的请求，也要保证 merged view 能响应；
2） 用 merged view 响应请求的开销不能显著大于 原有的候选mv 集合；

通过尽可能不修改 parent view 的结构来保证性质1；

通过合并后判断 merged view 的 size 不大于 原有的集合M 来保证性质2；// 感觉这一步不太说得通？ size 不大于 不意味着查询开销不大于？

MergeViewPair 算法用自然语言描述相当于：
合并二者的 projection column；
合并二者的 group by；
同时将二者各自的 selection condition 移动到 group by；
保留二者共同的 selection condition；
最后根据得到的 view size 筛选一下是否需要保留本次合并结果；

4.3.2 Algorithm for generation merged views

1）合并得到的 merged view 还会作为下一次合并迭代的输入，也许会被再次合并；
2）虽然算法的最坏情况会添加指数级别的 merged view，但是实验发现通常会提早结束（由于筛选；
3）合并顺序不影响结果


5 Trading Choices of Indexes and Materialized Views

跟 MV-FIRST 和 IND-FIRST 的两种方法进行了对比，（注意本文的方法是同时挑选出 MV 和 INDEX；// 不过具体怎么做的来者？ 如何评估一个 index 和 mv 能带来的增益？ // 似乎是通过 optimizer 模拟估计的增益，而这里如何进行评估的似乎就没有提及了；

5.1
首先设置 存储空间使用限制； 但是该给 mv 和 index 各自预留多少空间才能选出最优解？这取决于 workload（还是静态的

5.2 Joint Enumeration
文中宣称会比分开考虑 mv 和 index 更优啦，但是整体的思路还是很简单的贪心

6 Experiments

不过实验结果表明，比起以往的方法确实大幅度减少了运行时间；


