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

// model-free 
// model-based 都是什么？


abstract 

MCTS 需要大量 rollout， -> 应用成本高 （等下 rollout是什么，大量的随机模拟吗？

且难以并行，因为需要 node visit count 等统计数据来进行 exploration-exploitation 的选择

因此这里通过一系列能够监测 on-going simulation 的 statistics ，让 UCT 策略能够并行执行（？大概）

UCT: UPPER CONFIDENCE BOUND FOR TREES

1 Intro
// 没说啥


2 On the Difficulties of Parallelizing MCTS

2.1 Monte Carlo Tree Search and Upper Confidence Bound for Tree

介绍MCTS和UCT的概念（是不是可以抄到论文里 ^_^

2.2 The Intrinsic Difficulties of Parallelizing MCTS  

并发情况下，可能多个 worker 选择同一个节点（先到的 worker 还没更新 statistic），影响 探索 ；导致性能下降（虽然感觉最后都是会扩展出来的？

// 很像的B树并发控制
// 不过还是有所不同

3 WU-UCT

3.1 Watch the Unobserved Samples in UCT Tree Policy

对执行中的 worker 添加一个新的度量 O_s,然后在 Policy 根号内的部分加上 O_s；

3.2 System Implementation using Master-Worker Architectures

expension & simulation 比较耗时间，这部分可以并行；（实现也比较容易

但不同的 worker 需要使用最新的 statistics；这部分通过一个中心化的 statistics 实现；

4 The Benefit of Watching Unobserved Samples

对比了一下其他几种 并行MCTS 方法，
说WU-MCTS能更好的取舍 exploration & exploitation

5 Experiments

实验证明 WU-MCTS outperform other baseline
