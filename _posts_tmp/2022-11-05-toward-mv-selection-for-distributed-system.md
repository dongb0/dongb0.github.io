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
本文考虑使用 mv 来优化不同资源限制（如CPU，IO，带宽）下异构节点的查询和更新，这可能在大型企业有需求

在分布式场景中选择哪些视图进行物化是一个 NP-hard 问题，solution space 非常大，且非线性代价模型的，因此基本无法使用暴力搜索、贪心，等方法，目前所有的 mv 选择方法也不考虑分布式环境和更新开销。

本文提出一种算法框架，可以对候选 mv 进行限制，从而使得我们能够应用遗传算法来进行选择。

为了使遗传算法迅速收敛，我们基于数据库 tuning 的知识来设置初始条件（initial population），并且限制 solution space。

最后在实验数据集和真实场景中进行了实验评估，9节点24张表，可以在30s内计算得到 solution；对于 400 节点 1000 张表，可以在 15 分钟得到结果。


---------------
遗传算法啊，不是很想看

-------------------------
2022-10-22


1 Introduction

挑战
1）分布式视图选择是NP-hard 问题（不说分布式，一般的视图选择似乎就是 NP Complete）
2）分布式环境下，节点的资源限制不同，导致 cost model 是非单调的；此时贪心算法并不适用（但是98年的论文如何贪心的》
3）可物化视图的数量随着 节点数量、column 数量，join predicate 数量 等因素指数型增长，庞大的搜索空间使得 遗传算法 等无法直接应用

这里提出，算法要满足两个要求：
1）对不准确开销要能提供 robust result
2）支持灵活的 cost model

有不少分布式物化视图选择的研究 \[4, 10, 12, 25]，\[10, 11, 12, 13, 16] 但前一类没有考虑 update cost，后一类 only one querying node ？ // 只对1个节点支持查询、其他节点只是存储的意思吗？ ？是否要看一下其中一篇？

本文提出的框架简化了 solution space，使遗传算法能够应用；
1）将 solution space 表示为 bit 矩阵
2）提出让遗传算法迅速收敛的 mechanisms


1）首先使用 \[2] 中的方法选择 “costly” table； // 要去看2000年的那篇论文。。。
2）通过句法分析选择 mv candidates，分析 aggregation op，join，predicates，然后对这些 op 进行重新组合来生成 “promissing candidates”
3）使用遗传算法选择 mv 和用来物化他们的节点

遗传算法的初始化 population 的选择上，包含最小化 query cost 或者 update cost 的候选，然后开始遗传；
论文认为，遗传算法不能保证最优解，但是能够避免陷入局部最优的特性，适合用于解决视图选择问题；

场景是 RFID ？标签的识别？还是类似零售的场景？

sec 2 related work
sec 3 RFID scenario and motivation
sec 4 approach
sec 5 experiment
sec 6 conclusion


2 Related Work

相关工作分类：
1）集中式的，一个节点存储所有的 table，mv
2）半分布式的，table 分布式存储在不同节点，但是 mv 只物化在1个节点上
3）分布式，table 和 mv 可以存放在任意节点

对于1），文中认为 storage 是主要限制因素 // 真的吗，早在2000年前后的文章就认为存储空间不再是主要限制了

\[2] 用于在集中式关系型数据库选择索引和 mv，本文算法的初始候选 mv 的生成就采用了 2 的方法； 
// 因此有必要看一看 2

\[28] 扩展了 2 的工作，增加了水平分割，只物化经常被访问的行 // 是否太复杂了些？
概括来说这两个工作是采用贪心算法的？ 如果按照这里概括的说法 不断构建子问题的最优解的话；

一部分工作主要考虑 update cost 作为限制条件；
如 \[24]  ，但它的方法可能会陷入局部最优； 
\[3, 29] 也有类似问题

\[6,21,7,19，27] 等是针对 data warehouse 的 mv 选择，主要考虑不同的 update strategy

\[26] 采用了遗传算法，但是无法扩展到分布式环境下，（search space 太大了）

对于2），相关工作有 \[12,10,11,16,13,15,22]，但是并不打算具体看，如果以后有机会的话再说；

对于3），相关工作较少，\[4] 是这方向出现的第一篇 paper，针对 data warehouse的，不过算法应该也能应用在 RDBMS 上；但是它没有考虑 load 和 网络传输，也没处理 update 的限制；

\[25] 则采用了某种贪心算法，同样考虑的是 线性模型（而没有考虑 update 开销；

3 Application Scenario

是针对零售商的场景，有 ERP（Enterprice Resource Planning system），MIS(manufacturing information system), POS(point of sale), RFID设备，每个单位都能物化视图






