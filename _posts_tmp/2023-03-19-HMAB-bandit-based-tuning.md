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

// TODO：要去看：
// 1. 如何结合 优化器 和 执行数据 获取 PDS 的 nenefit
// 2. 如何生成候选
// 3. 这个方法的应用场景
// 4. 最后是看看算法怎么实现和运行

abstract



1 intro

两层结构的 tuning system，一层用来生成候选，第二层使用 MAB 选择 PDS

如何尽量少开销的获取 PDS （physical design structure) 的 benefit

contribution：
1. online learning method for tuning PDS（index 和 MV）
2. 支持 large action space 和 并行处理
3. 一种结合优化器的统计数据与运行统计数据，来减少 PDS 创建的运行时间 （减少 58%）
4. 在 dynamic workloads tuning 下性能优越（ 96% 的提升 ，比起商业的 SOTA 工具）
5. 性能优于 9 种 index tuning 方法 

2 Problem Formulation

他这里统计的是很多轮 execution，每轮过后可以重新推荐 PDS，并且创建

3 Bandit Background

3.1 A Simple Bandit Setting


3.2 Contextual Combinatorial Bandit Setting
跟简单 bendit 环境不同之处：
1）我们可以知道 arm 的cost
2）一次可以摇动 k 个 arm（选择一个集合来摇，当成一个 super arm
3）对于 super arm，每次摇动我们得到的是 k 个 arm 各自的收益


3.3 An Implementation
为什么提到 $C^2UCB$ ? 而且这个算法暂时也看不太懂啊


4 Design and Implementation


4.1 Overview
L1 选candidate,L2推荐并反馈 reward 给 L1

这种结构可以 增删 L1 中的 arm（也就是 candidate集合），而不影响 L2 的推荐

4.2 L1 and L2 bandit
也是基于 workload 生成 （index 的）candidate,可以减少候选数量

index 的生成是 基于 predicate 和 payload（应该就是投影部分？）的排列组合
mv 的生成，首先选择 frequent table subsets,
然后根据涉及这些 frequent table 的 query 生成 mv 首选（带或不带group by）
// 但是这个生成方法没太说清楚啊

4.2.4 Design of the Reward


4.3 Use of Optimizer to Reduce Exploration Check
哥们你也只能用 optmizer 来检查 index 啊，我mv咋办，那不还是得创建吗

4.4 Putting it all together

4.5 Implemmentation

5 Experiment

5.4 View Selection
跟几个贪心方法对比：
1）TopValue（获取最高的运行时间缩减
2）TopUValue（时间缩减/size 比值 最大的
3）TopFreq（选择最常被使用的view

5.5 Impart of Changing L1 Bandits
他们的架构下，改变 L1 不会产生巨大的性能都懂（架构到底啥样来着？ L1 选 candidate？L2推荐？

5.6 Impact of Hierachical Bandit Structure on Recommendation Time


5.7 Impact of Hypothetical Check
对于 index,能用这样的检查，缩短了 58%、25%、8%、34%的 索引创建时间（对四个benchmark（不被使用的索引不需要创建
