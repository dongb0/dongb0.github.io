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

在数据分析集群中，观察到许多任务之间存在重复计算的情况，这会对计算资源造成浪费，增加任务的执行时间；因而发现并处理这些重复计算的现象，是operator和user都关心的问题。但是现有方法并不支持（如此大规模）的workload。

本文将针对 large workload 中的 subexpression 选择问题，通过将任务的 common part 物化来提高任务的执行速度。文中将这一过程描述为 ILP 问题，并将其映射为二分图的标记问题。然后提出 BigSub算法，以顶点为中心的迭代式选择 subexpression 物化，并在微软的 SCOPE 中提供了分布式的实现，在实际系统中最多能节省40%的运行时间。

1 Introduction
现状是，分析型数据中心里，常常发现有大量（文中统计数据是45%）的任务存在相似性，（不过前提是数据中心规模比较大，数千用户每天会提交数十万个分析任务）；

因此考虑一系列优化措施：view selection、multi-query optimization、subexpression reusing；这些优化的相似之处在于：给定一组查询，寻找其中的相似部分，在给定的约束条件下实现最小化代价函数的目标；

但以往许多优化算法，应用场景（benchmark）中的查询数量为数十到数百个查询，这比 bigsub 要求的用户查询数量小至少3个数量级；

也有一些面向大规模集群的方法，如 nectar 
文献\[18] online caching 方面考虑实现 reusing（是否要去看一眼？
依靠 heuristic 来选择 subexpression 从而进行物化，虽然这样的方式不会受限于集群规模，但是依赖局部优化决策的方式也导致整体性能里全局最优解相去甚远；

因此需要全局优化的策略；这里提出 BigSub，主要关注的是 subexpression selection
周期性的对数千条查询构成的整体 workload 进行分析，选择其中可靠的 subexpression 物化；（因此可以算作是静态的

将 subexpression selection 映射为二分图标记问题；顶点表示查询和候选的 subexpression，边连接 query 和它的 subexression；

然后再划分为两个子问题：
1）标记 subexpresison vertex，表示它将被物化
2）标记 edge，表示物化的 subexpression 将被 query 使用；

同时为了将算法扩展到预期的规模，使用了一个 vertex-centric graph modle，迭代式的重复上述标记过程直到算法收敛

contribution：
*
*
*
* 提出减少 possible solution space 的方法，并且不会降低 solution 的质量；
* 在生产环境 workload 中实验表明，1）支持上万查询规模；2）优于启发式算法3个数量级；3）节省大约10～40%的运行时间

2 Problem Formulation

场景是：workload 中有些查询会周期性出现（monthly/daily）；考虑到 workload 相对 static，这里采用定期 offline 选择 subexpression 的方式；

2.1 Problem Statement

Subexpression Selection
目标与视图选择一样，都是从候选集合 S 中选出能够最小化 代价函数的 集合 s；并且要满足约束条件 C；

Definition 1（candidate subexpression）：假设有查询 q，称其对应的逻辑计划树结构为 t，t的任意子树 t‘ 都是 q 的候选 subexpression；

Candidate Subexpression Enumeration
选出所有可能的候选subexpression，用于后续的选择算法； complete subexpression strategy 会选择“所有”可能的subexpr，但这里只采用了优化器输出的逻辑计划；原因如下：1）更少intrusive，侵入性，也就是更容易实现；2）减少搜索空间；3）复用现有的plan signature 快速寻找 等价subexpr；4）利用之前的查询运行的统计数据

Utility of Subexpression


Subexpression interaction

3 Scaling Subexpression Selection

4 Optimization

5 Evaluation

6 Related Work
View Selection
在 data warehouse 场景下提出了不少 techniques： AND/OR graph（最老的、模型化为 state optimization 文献\[46]、和 data cube lattice（1996 文献\[22]

Subexpression selection
这算是 view selection 的子问题，view selection 可以选择那些没有物理计划对应的逻辑计划；但同样的 view selection 的 search space 就会变得很大；而扩展性是本文的主要考虑因素，因此选择的是 subexpression。
文献 \[44] 考虑云上执行环境中，同一个任务脚本里的 common subexpression；

Multi-query Optimization
MQO 与 view selection 类似，区别在与 MQO 只是执行过程中暂时的物化查询视图； 文献\[42] 使用 AND/OR DAG 在火山模型的优化器中完成 MQO 任务，但是没有考虑存储限制。
文献\[30]提供一种 subexpression 数量的平方 级别的渐进算法
文献\[6] 采用遗传算法
但这些算法并不能并行化；
在 mapreduce 场景中也有对 MQP 的研究
文献\[8,38,49]，但query的数量都很小（tens of

Reusing intermediate result
该问题研究重新利用那些已经被物化的中间执行结果 文献\[23,37] 
这对于大数据平台而言非常有效，因为：1）包含大量重复计算；2）优化用时相较于执行时长来说非常短；3）获得的性能提升很大
有 Restore、Nectar（引用见原文）和文献\[39]

Shared Workload Optimization
为一组query构造common query plan，使得它们能够共享

Large-scale Workload Analytics
vertex-centric programming model 在大规模图分析中十分常见。【若干个系统】



7 Conclusion



