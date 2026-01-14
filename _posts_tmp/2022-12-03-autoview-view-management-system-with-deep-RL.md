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
// TODO：记得看看 related work

// 论文中经常出现的 cardinality 是指什么？

// 实验的话，需要物化怎么样的一个视图？
// 1）可不可以申请一个企业版 OB 使用权，然后只跑 TPC-DS 的 benchmark？模型和视图就用算法跑好推荐出来的那些？手动创建一下？// 但是这样怎么跟传统的贪心算法进行对比？// 那贪心的同样用 python 推荐好，然后再手动创建视图。。。可以吗

// 如果 企业版不给用，那怎么办？云 OB？但是如果不能改内核代码，它能实现 rewrite 来利用现有的物化视图吗？ // 问一下师兄

// 2）或者只挑选能够支持物化的那些 query
// 3）自己实现剩下的 join（不知道够不够），或者询问 OB 是否已经实现，或者请求海康协助 QAQ 
// 最硬核的方式，就是自己实现剩下需要的功能了，这样子如果还有需要添加的要求也比较好把控

// 就是 python 训练好模型，然后跑算法推荐出来对应的物化视图

// 分布式的环境里有什么不同？

// 有没有可能不用实际在 OB 上实现？ 比如参考 1）



abstract

deep reinforcement learning,
storage budget,

看起来像是静态的

1 Introduction

四个模块：
1) MV candidate generation
2) MV Cost/Benefit estimation
3) MV Selection
4) MV-aware query rewriting // 啊这个没有搞过的东西 // 当获取新的查询，能够利用现有 mv 的话，要基于选择的 mv 重写该查询； 
// 看不懂他这里 rewrite 到底搞了什么东西

challenges：
1）mv 选择需要估算 mv 带来的增益，但现有方法不够？ // 文献\[1,12]
2）传统方法将 mv 选择看作背包问题，使用贪心算法，// 局限很多
3）query rewriting 也依赖于 cost model，但现有方法常采用优化器的代价模型，不能准确估计 mv 的增益

AutoView 使用 RNN 来处理 queries 和 mv 表征为 embedding ector，然后使用 RL  ERDDQN 来选择 mv； 也使用 ERDDQN 来选择重写 mv；

2 AutoView Overview

A. Problem Formulation
给定一组查询 Q，生成一组物化视图 V，使得：1） V 的大小在给定的存储空间限制内；2）使用集合 V 来处理 Q 的性能最优

// TODO：怎么重写啊？

Query Rewriting With MVs
给定视图集合 V 和查询集合 Q，选出一组子集 Vk 用于响应查询 Q，使得 Q 的查询性能最优；

B. System Overview

MV Candidate Generator
同样是来找 common subquery 来作为 candidate，
// 但是这里没有详细介绍怎么找，可能得去参考 2020年 那篇长文；
// 细节在 sec 3

相似条件的 subquery 会被合并，如
where country in (a, b) group by country 和
where country in (c) group by country 会被合并成
where country in (a, b, c) group by country

MV Estimation
该模块估算使用某个物化视图带来的增益（即节省的开销）

最简单的实现方式是利用优化器提供的估算功能；但考虑优化器估算的错误较多，这里使用了深度学习模型来进行估计，具体而言是 RNN 模型（后文将稍微详细的介绍

MV Selection
将该问题模型化为 Integer Programming？ （是不是少了个 Linear？
使用 RL 模型完成该任务；在 sec 4 中稍微详细介绍

MV-aware Query Rewriting
使用上面的 MV Estimation 模块选择出 mv 之后，在这里进行重写 // 有没有可能我不做重写啊？那我能不能进行实验啊？

C. Related Work

基于机器学习的方法有：
Bigsub \[4]
\[6] DQN 模型进行选择，但似乎是面向静态的？workload 变更时需要重新训练
\[12] DQN，workload 变化时同样需要重新训练

// 那么这里如何解决这一问题？

3 View Candidate Generation
难点在于如何找到 关键 的 view candidate；这里采用的（与之前类似）的方法是将 subquery 作为 view candidate；

有两种方法获取 subquery：
1）是搜索 SQL 中的 sub-expression，但这样的缺点是：a）subexpression 可能不是合法的 subquery；2）高收益的 mv 可能不会在 subexpression 中显示的出现
2）是获取 SQL 对应的 tree structure，然后选择最高收益的子树；但一条 SQL 可能对应很多执行计划，形成很多子树；这里选择的是 优化器 推荐的执行计划（文中考虑这样获取最高的收益 mv 的概率比较大）；

然后提出一种能够有效抽取 subexpression 并生成 mv 的方法；

MV Candidate Generation Framework：
首先抽取物理计划的树状结构，检测其中 common subtree：有相同的 join/selection/projection，且它们的节点也等价的；

然后合并这样的等价节点，计算 benefit（包含 common subtree 的 query 数量 与 估算代价的乘积；然后选择最高收益的 subtree 作为 candidate；

这里的主要挑战是如何高效检测 common subtree；

Merging Similar Nodes：
就简单举了个例子？

没有什么精妙的算法设计吗？

Merging Query Plan Tree：
将所有查询计划合并为 MVPP Multiple View Processing Plan 文献\[11]，是一个合并了所有执行计划的有向无环图，能够高效寻找 high benefit subtree？

合并完之后，统计每个节点在 图中（合并的？）频率，选取其中 benefit 高的（多少？）作为 candidate

4 MV Selection
类似之前的 ILP 定义

A. Benefit/Cost Estimation of MV Candidates

// 感觉好难做TAT

估算的开销是：

Wv = {f_v * (t_v - t_scan)} / |v|

f_v 是 merged graph 中 subtree（或者说 node）的访问频率
t_v 使用 Encoder-Reducer 估算的执行时间
t_scan 是扫描 result 所需的时间（通过 size * 读取每单位数据的 IO 时间
|v| 是 subtree 对应 result size 的估算值


B. DDQN model for MV Selection

还是有 ILP solver 的，但是用来解视图选择就很慢了；
这里计划使用 DDQN，但是有两个挑战：
1）对于不同的 workload，视图选择的参数是不一样的，难以进行表征；
2）难以对 MV 之间的关系进行编码；

对于 1）这里使用了一种迭代方法来选择 mv，以便将全局状态分成多个固定的子状态
对于 2）设计了一种新的表征方法，称为 Encoder-Reducer DDQN，由六部分组成：

Environment：存储全局 MV 选择的状态，并在迭代过程中计算总收益；Environment 将全局状态看作一个二分图，左边是查询 Q，右边是视图 V，节点之间的连线表示使用 V 来回答 Q；

Agent：两个神经网络和一个 experience replace 机制；神经网络是拟合 action-value 函数 W(s, a), 表示在状态 s 进行 action a（比如将 e_ij 从 0 变为 1）的 feedback G；G 与 benefit 正相关； 

Reward
为了实现 G 与最终收益正相关，这里定义 reward 为每次行动之后 benefit 的变化；G_0 是没有 decay 时能获取的最大收益，因此加入 decay 之后，行动次数越多 feedback 就越少；

// 看看 decay 是什么，是不是每次行动需要消耗的代价

Action
从所有边中选出一条边 e_ij，然后询问 agent 是否采取该行动；如果 agent 判断为 yes 则将 e_ij 标记为 1；

Policy
就是选择能够获取最高 feedback 的行动

State
通过 Encoder-Reducer 模型得到的
semantic vector

vector 包含的信息（如 MV 的结构信息）可以用来判断两个 MV 是否在 1 个 query 上发生冲突；通过 row witdh 和 cardinality 来估计 MV 的大小；

Solving Procedure
agent 按照 Policy 采取行动（action） 来与环境（environment） 交互，并且检测环境的改变，如果观测到 selection state 收敛或者达到最大迭代次数，则将全局选择状态中，收益最高的保存为 final solution；

Training
ERDDQN 依赖于 experience replay
// 这是 DQN 的什么机制吗？

需要建立 experience pool 和 sample experience tuples（s_t, a_t, S_t+1, R_t+1)，

s_t 表示当前状态，a_t 表示 s_t 状态下选择的行动，S_t+1 表示经过行动 a_t 后的下一个状态，

W*(s_t, a_t) 是 Q-network 的估计值，随着迭代将不断接近真实值；

下面给出了一个更新 Q-network 的公式，// 这里略去不写


5 Experiments

IMDB 数据集，JOB workload

对比算法
贪心
TopValue
TopUValue
TopFreq

BigSub

然后是文中模型调整一些结构得到的变体；
