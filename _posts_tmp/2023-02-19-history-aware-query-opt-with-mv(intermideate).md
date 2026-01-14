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

2014

// 问题：如何判断一个 中间结果 将来会被用到的可能性？

abstract

提出一种集成在优化器内的、能够利用历史查询的中间计算结果，进行物化，来减少未来查询的开销

1 Intro

这里的中间结果，是会放到一个 pool 中（那么就应该是类似动态的

贡献：
1）
2）提出一种能够利用中间结果的查询优化模型
3）selecting and pooling mv的策略
4）


2 Releated Work


3 System Overview

有两个缓存池：
History Pool
最近执行的查询

View Pool
已经创建的view（当然是当作table

当新查询 Q 到来时：
1）首先Hawc会利用 mv 来优化Q（改写查询；其次，Hawc在评估某个执行计划 p 时，如果p能够对 History pool 中的查询带来增益，那么这个增益也会被 p 考虑在内
2）执行查询 Q 之后，Hawc通过integer opt program来寻找使得 history query 查询开销最小的 views 组合进行物化
3）更新 history pool

// 注意，这里特意提到的应用场景是：hadoop节点中 CPU/RAM 等资源紧缺，但存储空间剩余很多 （1.6T

// 不过我还是有问题，如果 workload 一直变化不重复，那么这个方法还有用？
// 但是好像其他的方法在这种情况下也不好使；而且一般的workload查询之间是存在一定关联的，毕竟表就这么多（一般就几十张？
// 感觉到这里就可以写综述了呀，后面还用看吗？
比如具体如何选择 views

A. Costing in the Hawc Optimizer
查询优化同样分成两个 阶段：logical phase , physical phase

判断利用/存储 中间结果 的步骤发生在 逻辑阶段

每个逻辑计划 自带 一个执行代价，定义一个 c（可能是tuple数量，也可能是字节数、RDF数量等
同时 Hawc 认为 cost 是 additive，

认为cost可加之后，替换新的 plan,代价也就可以由替换后新查询 p' 的 subplan 计算得到

B. History Pool
一系列查询的集合，每个查询条目包括：
1）权重w,随时间衰减，表示该查询执行时，对历史查询的增益（所以这里是认为越旧的查询，对未来增益越小
2）一组查询计划，用来评估新查询的执行计划的效果；除了实际执行的 plan 之外，一个查询的其他等价 plan 也会被加入pool中（同时因为每个 plan 都包含对应的 cost, sub plan也是，因此在计算 utility 时，很容易通过 plan 得到。

C. View Pool
pool中每个V包含：
1）plan p
2）hash h（为了快速搜索某个视图
3）用来描述 V 的 statistic delta
4）做视图 table scan 的 cost vector 


5 Query Optimization in Detail

Hawc 的 优化器，除了选择开销最小的plan 之外，还需要评估 alternative plan ，看能否通过物化 intermididate result 来获取有用的物化视图；

这个过程不仅需要对 alternative plan 本身的 cost 进行计算，还要计算 它对于 history query 的 hypothetical cost；
即，物化这个plan 的 view,能够为全部的 history query 带来多少增益；


B. Cost with the History Pool

对于一个 history query（包含一系列 plan）
需要对每个 plan 的 cost 进行评估，选取cost最小的

这里评估则是从 plan 的 root 开始，然后贪心的选择 cost 最小的 subplan


6 Update The Pools


