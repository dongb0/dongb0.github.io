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

abstract
adaptive mv and eviction policy

无需精确的代价估计、手工定义的启发规则  

优于启发算法；
可以应对动态workload；
适用于不同workload和 skew
// 总结：也就是动态的视图选择，同时使用异步的强化学习，看看怎么实现

在 Spark-SQL 上实现的；

1 Intro

分析现有动态 view selection,是基于过往的 query,且假设未来的变化并不大

通常使用一些启发式的策略，如LRU/LFU；以及有些基于cost model的方法

\[28] 是另一篇 2014年的 动态选择 文献，考虑要不要看下 HAWC  （已经下载

2 Background

3 System Architecture

3.1 Decision Model (问题定义

目标同样是最小化查询开销（这里是 latency = cost_create(mv) + cost(q_i)
同时有存储空间限制，以及这里要学习的还有 eviction policy

3.2 Architecture



4 Learning Materialization

4.1 The One View Problem


4.1.1 Opportunistic Setting

不太懂，好像是要寻找 物化 的时间点？

4.1.2 Counterfactural Runntime Experiments

不就是要估算 benefit,需要一个不使用 mv 的 query cost 和一个 使用 mv 的 cost 吗，有必要还单独搞一个实验 用一个章节来写吗 <_<
 
4.1.3 Asynchronous RL Algorithm

// 通过实际运行 SQL（当系统空闲时）来获取视图的增益
介绍了一下编码，好像也没什么东西

5 View Eviction

5.2 Algorithm
维护一个表格 记录 view 的 credit,每次有新查询时都会更新，没有被引用的视图，credit 会衰减；需要 evict 时，从表中选择 credit 最低的驱逐。

6 Experiments
与 LRU/LFU进行替换 以及 使用启发式方法进行选择和替换的 两项工作进行了对比；

简要概括结果：在考虑模型训练时间的情况下，也与这两项 启发式 的性能接近；甚至优于

