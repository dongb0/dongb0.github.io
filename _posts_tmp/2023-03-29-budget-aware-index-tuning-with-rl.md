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

// Q：如何限制 what if call 数量；根据什么条件来限制？
// 如何满足实现？

abstact
即使是 what if call 也算高开销了
提出一种 MCTS 的 index tuning 方法


1 Intro

index tuning 是关系型数据库中重要问题，

index tuning advisors 接收 workload 作为输入，在 index 个数或者 storage 限制下，推荐出一组索引

index tuinin tool 分为两个模块：
1）index generation
2）configuration enumeration

其中 2）需要调用优化器的 what if call 进行分析
但因为 what if 是 resource intensive，所以常常通过 cache 来复用 calls
// TODO：paper Anytime Algorithm of Database Tuning Advisor for Microsoft SQL Server.

云上数据库中，index tuning 通常在生产服务器中完成（使用备份也是需要成本的
// TODO：paper Automatically Indexing Millions of Databases in Microsoft Azure SQL Database
// TODO：paper Constrained Physical Design Tuning

// 一般索引也就推荐 20 个吗？ 那我这感觉差不多了呀

what if call 每个1s, 5000个就100分钟左右


本文工作是面向 offline 的，workload 恒定；主要创新点在于 限制 what if call 的数量，变相控制 tuning 的时间；

在 limited budget 限制下，significantly better；（基线是什么？

2 A brief overview of index tuning


3 Budget-aware Index Tuning

3.1.1 cost derivation
似乎是某种估计开销的方法？不需要实际的 what if calls
（可以通过少量的 what if 来评估

3.1.2 The Greedy Algorithm with Derived Cost


3.2 Budget Allocation in Configuration Search
定义一个矩阵，表示分配 what if call 分配给
（query，configuration）的分配情况


4 Budget-aware Greedy Search


5 Budget Allocation With MCTS

5.1 An MDP view of Configuration Search
这里奖励是 cost_with_idx 占 不使用index 时 总开销的百分比 


