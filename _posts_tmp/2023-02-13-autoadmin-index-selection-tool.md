---
layout: post
title: "[]"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - 
---

// 这个可以写进index selection 章节里

abstract 


1 Intro

1.1 Contribution

1) end to end tools for index selection(on SQL Server)
2) novel search technique for index candidate genereation
3) interative search procedure for index selection

可能是较早（可以说首个吗？）提供大量实验的paper

search time 缩短四到十倍，同时查询性能没有明显降低

2 Overview


3 Cost Evaluation

这里已经有 what if indexes 了，但是还是想减少调用优化器的次数；

提出 atomic configuration 概念，即如果workload中有一个以上查询使用了某个 configuration 中所有的 index,则说此 config 是 atomic.

因此若有 M 个 config，不需要执行 M 次，只需要执行其中 包含所有atomic config 的 M' 就足够

3.1 Deriving Cost of a Configuration from Atomic Configurations

如果一个 config C 不是 atomic,则 atomic config 是 C 的子集。那么执行查询 Q 时，优化器将会选择能够最小化查询开销的 atomic config  C_i（理论上），因此我们可以将 Cost(Q,C) = minCost(Q,C_i) // 对于select/update
如果 cost(Q,C_i) 代价已知，则此时获取 cost(Q,C) 无需再调用优化器


4 Candidate Index Selection

首先，对每个query独立生成candidate,并把 能够成为一个或多个query的 best configuration 的索引，作为 final configuration；（有点绕，不过是这个意思）

作为候选的index,需要是每个query的best candidate；这里认为如果一个index如果不是一个query的 best design,那么也不太可能成为整个workload的best design
// 这个好像并不一定，能举个反例子吗

// 如A、B两查询，index i1 能为A带来50,B带来50增益；同时存在index i2 只能给A带来75,index i3 只能给B带来75；对于索引，可能出现这种情况吗？ 
// 感觉似乎是有可能的

总而言之，现在问题在这里转变为如何为单个query挑选出best design。

将workload拆分到单个的query,然后从其中（starting candidate 是所有可建立索引的 column）选出 k 个作为 configuration（通过枚举？ // 在 sec 5 中展开介绍 // 并没有
// 尼玛的，我好像懂了，他产生候选的方式他妈的居然使用索引选择的模块来做的。只不过这时候的输入候选是由所有的 indexable column 产生的，所以不会无限递归。并且因为这个索引选择的算法是枚举+贪心，所以对于index少的查询也蛮容易运作的。


5 Configuration Enumeration
穷举能保证最优解，但不适合
要从 n 个基础候选 index 中选出 k 个索引，但搜索空间太大无法完成（n=40, k=10，C_n^k=C_{40}^{10}=847660528）

这里采用Greedy(m, k)，先枚举选出m个(m<=k) 最优，然后贪心搜索得到剩下的 k-m 个索引 

贪心的部分怎么考虑所有可能的index？

有文章表明使用贪心方法进行 index enumeration 可能 arbiturarilly bad，但文中说实验结果还不错（只要m选得比较小） 


TODO：
syntaticall analysis

[1988 TODS] Physical Database Design for Relational Database
[1976 SIGMOD] Index Selection in a Self-Adaptive Data Base Management System


