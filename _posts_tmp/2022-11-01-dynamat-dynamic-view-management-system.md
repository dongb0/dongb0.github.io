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
// Q1：Data Cube Lattice 是什么呢

// Q2：这个 DynaMat 不需要 lattice 是吧？//好像还是要的？用来找 f-father


abstract 

数据仓库中自动选择和管理物化数据的工具很重要；

本文提出一种能够动态物化数据来匹配工作负载的系统，DynaMat，它也将维护限制（时间限制和空间限制）纳入考虑。

DynaMat 会在运行过程中根据查询，在给定的 maintenance window 内调整更新物化视图；

本文将 DynaMat 与一个静态选择算法进行了对比，在 DXetailed Cost Savings Ratio 的度量标准下，DynaMat 表现更为优异；

1 Introduction

文中提到 \[HRU96, GHRU97, Gup97, BPT97, SDN98] 这些工作都是静态的视图选择，通常工作在 空间限制 的约束条件下；

不过主要问题是，静态的视图选择 可能会 outdated（// 确实存在这一可能，不过不知道现实场景中，这一可能出现的频率多高？需要工业界的数据支持啦，学术研究不搞这些）

注意：微软的 AutoAdmin 能提供类似功能？ 在索引选择里面也看到了；

Another drawback of static view selection：没有调整 wrong selection 的机会；

随着磁盘价格/每单位存储空间 的下降，存储空间限制通常不再是问题了；相反，是视图的维护时间，限制了我们不能过多的物化； // 一些研究 \[Gup97, BPT97] 考虑了这些问题 //还没看，还有另外的几篇 \[TS97] \[DDJ+ 98]
当然它们都没有考虑动态的视图选择，只能工作在静态环境下；

主要思路：不要浪费（一次查询）的结果，尝试重复利用，来将查询的计算开销分摊到其他查询上；

novel goodness measure，


sec 2 system overview
    2.2 & 2.3 reuse store results
    2.4 maintenance problem
sec 3 experiment
sec 4 conclusion

2 System overview


View pool V 用来存储物化结果；

1）on-line phase：
Fragment Locator 判断缓存结果是否能被使用。需要代价函数评估？ // why？
Director Index 用于支持 V 中的 sub-linear search
如果 V 中没有缓存的结果那么走常规路径，基础表+索引，来获取数据； 但是计算完数据之后，还是会通过 Admisison Control Entity 判断是否需要存储这些数据到 pool 中；
目标就是让尽可能多的数据能够从 pool 中获取到；   // 难点在于：如何判断这些缓存的数据可以重复使用？

2）update phase：
当 data sources 更新之后，更新 pool 中物化的数据；
文中假设 更新是 **离线** 的，这一过程中不允许执行 queries； // 这是很大的影响啊？
在 admin 设定的 update window W 时间内，要完成视图的更新（所以一些无法在时间限制内完成更新的视图会被丢弃


2.1 View Pool organization

View Pool 似乎用了额外的硬盘来物化数据；

[BDD+ 98] ROLAP，以 relational table 的形式存储 summary data；
问题：数据量大导致 summary table 扫描费时；

所以通常 relational table 和 indexing 并不足够支撑批量的增量更新操作；

提到 CubeTree \[RKR97] 提供存储和索引 // 所以如何做到？ // \[KR98] 中用 Cube Tree 存储 summary
和 
chunk files \[SS94, DRSN98];

整体来说是通过 View Pool 的维护来满足 time bound 和 space bound；
time bound 通过再 udpate 阶段丢弃无法在时间限制内完成更新的 视图 实现；
space bound 则是当 pool 到达空间限制之后，开始采用 repacement policy 来替换视图（如 LRU，FIFO？就这？还有一些启发式策略，就这？

2.2 Using MRFs as the basic logical unit of the pool

Multidimensional data warehouse (MDW) 是啥？

// The lattice \[HRU96] representation of the Data Cube
也许是提出 aggregation lattice 的文献，如果后面还是理解不了这个概念就去看看

其他视图选择算法 常常使用 这一表示，（能转化为 图 中 的优化问题或者寻路问题？



如果 V 中缓存的 单个 fragment f 不能计算得到某个 查询所需结果，那么大概率就无法通过 V 来计算得到 结果（所以需要走常规 基础表的查询路径）


根据上述 推论/结论，如果 V 中物化的 fragment 太小，那么也会降低能够从 V 响应查询的概率；因此需要 组合（combination）机制 （合并某些个 fragment // 但是这不就破坏上述条件了吗

// 那么如何限制 fragment 的数量？ 用|V|表示，后续将证明 计算和更新的开销 大约为 k|V|^2，k为常数

2.3 Answering queries using the Directory Index

给定 MRF f 和 查询 q，当且仅当 q 需要的每个 Ri。在 f 中都有对应的 range，同时每个 Ri = ()，f 中对应的 range 要么为空，要么覆盖整个 range domain？（这一点是为什么？只有一部分不行？），此时称 f cover q；

DynaMat 使用 Directory Index 来避免搜索每个 fragment；
// 又涉及到 Fig 4 里的 Lattice 了；好像真的需要看

lattice 中的每个节点都包含一个 index 来检索是否在 pool 中有对应的视图；

 DynaMat 使用 R tree 来实现索引的； // TODO： 需要再了解一些 R tree 吗，已经记不太清了；

在 V 中检索有三种情况：
1）恰好存在 包含 q 的 fragment，则直接返回查询结果；
2）如果不存在，则根据给定的代价模型，选择 best candidate 来计算 q；
3）如果没有 fragment 可以用来计算 q，则传给 warehouse 通过基础表进行计算；

1）和2）都需要先经过 Admission Control Entity 计算

V 中的 MRF 数量通常在 数千 这个数量级，因此可以假设 Directory Index 可以存放在内存中；
对于无法完全存放在内存的场景，可以考虑使用 R tree 的 packing 算法来压缩数据；

2.4 Pool Maintenance

我们需要定义一个 goodness 度量来 选择/替换 视图 
文中经过实验后，选出4个指标作为度量，能得到较好效果，分别是：

1） fragment 上次被 query 访问的时间
可以采用LRU

2） fragment 被访问的频率
可以采用 LFU

3） result 的 size，以 disk page 为单位
SFF，Smaller Fragment First

4） 当 fragment 不在 pool 中需要重新计算时的开销 
  goodness(f) = (freq(f) * c(f)) / size(f)
c(f) 是重新计算 f 的开销；可以用 Smaller Penalty First 策略；

2.4.1 Pool maintenance during queries
如果 pool space 足够，那么每个查询的结果都会被储存下来；

但如果空间不足，则取决于使用上述哪一种度量，来使用对应用的替换策略；
goodness 较小 的 fragment 会被替换；

2.4.2 Pool maintenance during udpates

\[AAD+ 96, HRU96, ZDN97]  fast computation of Data Cube aggregates
// TODO：看看到底这些聚合的数据怎么样算得比较快？

基础表的数据更新之后，Pool 中存储的数据也需要更新；

\[GMS93, GL95, JMS95, MQM97, RKR97] 增量更新 grouping 和 aggregation queries


文中假设能够得到 base data 的差分数据，或者 log file；并假设 aggregate function are self-maintainable；
以便能够根据 old value 和 changes 来计算得到 new value

**Computing an initial update plan**
目标是找到一个能够在 udpate window W 内（也就是更新时间限制）更新尽可能多 fragment 的 update plan；

为了减少查询每个 fragment 需要的 delta 数据时间，这里在预处理步骤中，将所有的 delta 提取出来存放为一个独立的视图 dV，称为 Cubetree；以便针对 multidimensional range query 能够快速检索

两种更新策略：
1）从 dV 中查询 delta 数据，然后增量更新 f；
2）对于从其他 fragment 计算得到的 f，则 cost 是 f father 更新之后重新计算 f 的开销；

系统用这两种策略计算开销，并选择其中更小的一个作为实际的更新策略；
缺点是：这样并不一定总是最优的 plan；比如如果 f1 紧跟在 f 之后被添加到池中，如果 f 的 father ？？//没看懂
// 也许采用 eager policy？ // 实验说 lazy 的性能差别不大

但 lazy 策略能够把 replace 和 makeFeasible 的复杂度从 O(V^3) 将为 O(V^2)


**Computing a feasible update plan for a given window**
总维护开销 UC(V)= 每个 fragment f 的维护成本之和

如果这个开销大于 update window W，那么就要选择不物化哪些 fragment；但是驱逐一个 f 之后，要考虑 f-child 的维护是否应从 f-father 还是 直接从 dV 增量更新 更小；


首先保证没有单个 fragment 的更新开销大于时间窗口（初始时直接丢弃）；
然后如果仍然大于 update window，那么通过计算 goodness 和相应的替换策略选择 fragment 丢弃，直到总维护开销小于 update window 为之；

计算一个 fragment 的 U-delta(f) 根据公式（在文中，需要遍历 f 的所有 children 计算 children 的新开销应从 dV 中选择还是 从
 f-father 中
 
 因此驱逐 k 个 fragment 的开销为 k|V|；最坏情况需要驱逐大部分 fragment 时则复杂度变为 |V|^2


// 总结来说，这里的要满足时间开销的方法其实就是穷举；

3 Experiments

prototype 验证算法；学到了；

比较的度量 Cost Saving Ratio

求和i ci * hi
————————————
求和i ci * ri

ci 是不使用 cache 时 qi 的执行代价（或者说延迟？，hi 是 qi 的缓存命中次数， ri 是 query 的总次数

不过在本文中，ci的定义略微需要修改：

如果 pool 能直接响应请求，则 节省的开销就是 ci；
如果查询 qi 需要通过其他 fragment 计算得到，此时的计算开销是 cf，那么节省的开销是 ci - cf；

// TODO：Q 这里有提到使用其他 f 计算新视图的方法吗？上文不是说如果没有 fragment 能直接返回结果，最后就认为不返回吗？


3.1 Comparison of different goodness policies

SPF 一般的表现比较好； 在实验环境里，一般能达到50%的 DCSR，也就是大概能比直接查询 MDW 节省一般的开销；

// 一般的数据仓库不实现这种功能吗？类似查询缓存的东西；

3.2 Comparison with the optimal static view selection

n 维 lattice 有 2^n 个不同的视图；静态 view selection 的 search space 就有 2^(2^n)，如 n=6,就有 2^64 多个；无法穷举


