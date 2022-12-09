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

// Q1：那么以往根据增益来选择 mv 的方法是如何获得 mv 的收益的？// 通过优化器估计吗？

// Q2：为什么可以是线性规划问题呢？在什么的约束下？这好像不是传统的视图选择问题？ 没有存储空间约束和维护开销约束？

// Q3：预测 mv 开销和收益的准确率如何？原本的开销是通过优化器收集的统计数据得到的，现在呢？ // 现在这篇文章连收益都是通过模型预测的，好处是？准确率比起原有方法呢？

// Q4：传统的 view selection，在给定 workload 下，如何选取 mv？
// 好像是搜索空间巨大的，把所有 查询 中 访问 table 组合 的子集 作为搜索空间；
// 那么的机器学习方法中，有什么创新？现在看到的好像是把 公共子查询 作为物化备选；这样搜索空间岂不是一下子就小了很多？

// 但相应的限制条件也没有传统方法那样考虑 存储开销和视图维护开销 了；这里的限制条件是子查询不重叠？ 不过物化过程的 overhead 倒是考虑在内了（计算 utility 过程中

// 对于不同数据集

// 从 workload 中提取公共子查询进行物化，减少重复的计算开销；

abstract

许多现有 mv 选择方法仍然需要 DBA 手动生成；这在大规模数据库中是难以扩展的；

因此本文提出一种自动生成 mv 的方法，能够选择 subqueries 中收益最高的部分生成 mv 来减少查询开销；

两个挑战：
1）如何评估 mv 对一个 query 的收益
2）如何选择最优的子查询来生成 mv？

对于1）我们通过一种基于神经网络的方法来进行评估，
对于2）我们将此问题模型化为 ILP（Interger Linear Programming），设计了一种 iterative 优化方法来选择子查询；但此方法不能保证收敛，因此增加了 MDP 马尔科夫决策和深度学习模型来解决这一问题，将本文方法的性能比起基准提升了 28%，8%，31%；

1 Introduction

OLAP 中分析查询通常会包括重复的子查询，这些 redundant 可以通过物化视图来减少重复计算的开销；

\[20,3,31,34] 看看这些文献都做了什么；
\[20] bipartite labeling problem，用一种迭代优化的方法来解

两个挑战：
1）难以评估某个mv对于某个 query 的具体收益；正常而言我们是通过 query 使用一个 mv 后减少的开销来衡量该 mv 的收益，但是我们无法对所有的 mv 都重写 query 并执行来得到实际的增益 // Q：那么以往根据增益来选择 mv 的方法是如何获得 mv 的收益的？// 通过优化器估计吗？

因此文中提出一种 RL 的评估模型；将抽取两类信息：1） query/view plan；2）相关 table；
然后将这些信息分为 numerical 和 non-numerical，并转化为 hidden representation；喂进评估模型中；

其次，为了能够自动选择的子查询来进行物化，文中将该问题转化为 Interger Linear Programming 问题，并通过一种迭代优化的方式来得到渐进最优解，避免了计算实际最优的计算开销；不过该方法不能稳定收敛，BigSub\[20] 中解决该问题的方法是禁止将已选择的 subquery 再转为 unselected，但是这样会退化为贪心算法而得到较差的结果；

文中的工作是通过 RL 将该问题转化为 马尔科夫决策 问题，然后应用 深度强化学习模型 DQN 来得到一个稳定的结果；

贡献：
1）定义问题为：？同时提出一个机器学习的系统来解
2）
3）定义子查询选择问题为 ILP 问题，并设计一种 RL 方法，能够稳定收敛
4）实验，性能得到提升；


2 Preliminaries
A.Subquery and Cost

Subquery

Cost：比如响应一个查询的CPU消耗和内存消耗等

B. Materialized View
MV Overhead
包括 space overhead 和 计算开销；

space overhead：
mv 存储所需的 空间 u 单位（这里是字节），每单位费用 alpha；那么空间开销就是 alpha * u；

total overhead：
存储视图的开销 加上 计算子查询的开销；

Benefit of a meterialized view：
收益的计算方式：直接执行查询 q 的开销 - 使用视图 v 执行查询 q 的开销；

Benefit of multiple materialized views：

有些视图（subquery）之间可能是“重叠”的，

因此定义了 
Overlapping Subqueries:
我们说两个子查询“重叠”(overlapping)，当且仅当他们的查询解析树是有公共的子树；

如果给出一组查询 Vs，他们之间没有重叠，那么计算增益时，可以直接累加各个视图的增益；

Utility of multiple materialized views:
workload Q 和一组物化视图 Vs 的 utility（也许翻译为利用率
定义为 Vs 的总增益减去 构建 Vs 的总开销；

// mv 的开销也用深度学习模型来评估啊？ 这，，准吗

C. Materialized View Selection
给定一个 workload Q，目标是选择一组最优的子查询进行物化；

因此这里把 Materialized View Selection 问题模型化为一个优化问题，期望使得 Utility 最大化；

这个问题实际包含两个优化目标：1）选择最优的子查询来生成 mv；2）选择最优的 mv 来响应查询（并且要能够满足 重叠子查询 的约束）；

如果我们能够搜索所有子查询的子集并计算它们对应的增益，那么当然能够找到最优解；但是实际场景下因为搜索空间非常大，暴力搜索往往是无法实现的；因此文中提出了一种强化学习模型来解决这一问题；

3 System Overview

系统分为3部分：预处理、在线推荐、离线训练；
预处理阶段将 workload 中的 候选子查询 抽取出来；然后根据两个在线推荐模型，选出 收益最高的 子查询生成 mv；而这两个模型是需要离线训练的；
最后，将使用这些 mv 重写查询；

Pre-process
这一流程由三块组成：subquery extractor, equivalence detector, subquery cluster；
文中首先用 subquery extractor 将 query 解析成逻辑计划，然后将 聚合、连接、投影 看作子查询/subplan；

然后使用 equivalent detector 推断两个子查询是否等价，收集其中等价的 pair；
// 用了 \[45] 中的方法，// 看不看再说吧

最后将等价的 subquery 构成 cluster，并挑选其中开销最低的作为候选；

文中通过维护对每个 query 维护一个 cluster ID 列表，通过这个 subquery list 来判断两个查询是否由重叠；


Online-recommendation
两部分构成：cost/utility estimator，view selector
cost/utility estimator:难点在与估计使用视图 Vs 下查询 Q 的开销；
这里就设计了个深度学习的模型来评估，只是介绍架构，具体实现没有提；

view selector:
设计了 RL 的视图选择模型，基于上一部分计算的到的 utility 来进行挑选；

Offline-training
training 的数据来自 metadata database；
查询引擎可以收集 query plan、view plan、table info（i.g. size)、cost of rewritten queries；

对于 cost estimation 模型的训练：
收集 view plan、query plan 和 相关 table 信息作为特征，将 actual cost of rewritten query 作为目标来训练模型；

对于 view selection 模型：
首先计算 查询 和 视图 的实际收益，然后使用这一收益得到 intermediate rewards 用来调试 RL 模型；


4 Utility Estimation
首先描述跟 cost 相关的特征，然后介绍模型

A. Feature Extraction
模型需要特征包含：query/view plan 和 相关的 table

特征的提取，其实是把 一个（子）计划转化为一个序列，如 S1 select user_id, memo from tuser_memo where dt='1010' and memo_type='pen'; 在这里会被转化为 \[Filter, AND, ED, dt, '1010', EQ, memo_type, 'pen'] 的序列（实际是从执行计划转化过来的而不是直接从 SQL

而后每个查询可能包含若干个这样的序列，因此最后得到的特征将是一个二维的序列矩阵；

Associated Tables
从 metadata 中收集相关的table 信息，包括：
input table 的schema；
input table 的统计数据；

// 这些特征还可分为 numerical 和 non-numerical 两类
属于 numerical 的有：table 的统计数据
non-numerical： plan sequence 和 table schema

B. Wide-Deep Model

1）Model Architecture：
模型包含两部分：a）wide linear model；b）deep model

linear model 学习输入的线性关系，deep model 学习非线性关系；

对 numerical feature 进行标准化（来消除特征之间数量级的差异）得到向量 Dc；
对于Dc 使用 affine transformation 来转化为 Dw 作为 linear part 的输出；

对于模型的 deep part，文章首先设计了一个 编码模型，来将 schema 和 plan sequence 编码为向量 Dm，De；然后与 Dc 拼接起来，使用两层 ResNet（两层全连接层 和 两层 activation layer） 进行学习；

最后使用一个有两层全连接层和1层activaton 的 regressor 合并 wide 和 deep part 的输出，得到最终的预测开销；


2）Encoding Feature：
(为了能够对 non-numerical 特征进行学习，当然需要对非数值特征进行编码)

需要编码的 ：Keyword，String

Keyword
使用独热编码（one hoc）转化为向量，维度为 keyword 数量；但是这样的编码方式矩阵太多稀疏，因此采用了一个 Keyword Embedding model 来将其转化为稠密矩阵 // TODO：具体怎么做？

String（是 Fig 6 不是 Fig 8,笔误了；
对于查询中出现的 string，文章采用的方式是将每个 string 看作 char 数组，数组中每个字符 表示为 128维 的 one-hot 编码；然后通过 embedding 转化为稠密编码矩阵；
最后通过一个两层的 CNN 转化为新的目标向量；

Encoding Query/View Plan
对于 query/view plan 得到的二维序列，文章采用了 LSTM 模型来进行处理（就当作NLP了

该二维矩阵的每个元素就是一个 keyword/string，就用前面提到的方法来进行编码得到一个向量// TODO：Q：为何文中说每个 keyword 得到的是一个 code？而不是 vector？；

最终再使用两个 LSTM 转化为最终的目标编码序列；
// 到这里了还是在做 estimate 的模型；甚至还没开始训练

Encoding Table Schema
schema 看作若干个 keyword 的序列，使用 keyword 的处理方式编码之后，通过 Keyword Embedding 转化为稠密向量；（再通过池化将向量序列转化为一个向量；

3）Model Training：
模型有五个部分，训练目标是通过训练数据来习得这五部分的参数；
输入包括查询q、视图v、table info t以及对应的开销 A；

// 感觉基本是在解释伪代码，大概步骤是特征标准化、拼接成向量之后，用这些向量来计算目标 cost，然后不断迭代调整上述五个参数；直到收敛

5 Materialized View Selection

文章将视图选择问题模型化为 ILP 问题，然后提出一种迭代优化的方式来获取渐进最优解（避免搜索实际最优解的昂贵开销）
但是这种方法通常难以收敛为全局最优解，因此采用了 MDP 和强化学习的方式来解决收敛问题；

A. ILP Problem and Iterative Optimization
1）Rewriting Problem：
这里的问题定义是：i）从候选子查询中选出最优的子查询进行物化；ii）对每个查询 q 选出最优的视图集合 Vs，且需要满足 overlapping subqueries 约束；

其中 ii）可以看作 ILP 问题，看看文章怎么解决；
// 这是什么问题？为什么问题变成这样了？

思想上还是要满足查询使用视图 Vs 后 的增益最大；
约束条件是子查询不重叠 // 但是 formula 1 没看懂，为什么啊？ // 就是如果 Yij 被选来查询，那么不会有其他重叠的视图同样被用来回应查询

要求的就是 Zj 的集合，来寻找需要物化的视图（Zj 是一个 bool 值，表示子查询 sj 是否被物化） 以及 Yij 的集合 （表示视图 Vsj 被用于相应查询 Qi，即求用于响应/重写查询的视图集合）

2）Optimizing Iteratively
该问题对于 large workload 还是难解，这里考虑的方式是分别优化 Z 和 Y；即先将 Y 看作常量来优化 Z，然后将 Z 看作常量来优化 Y；

// 这，，，可行吗？// 优化问题完全不懂

Initializing Z and Y
随机初始化 zj 为 0或1 并记录此时最大增益；然后初始化 Yij 为 0 或 1（要满足约束条件；


Optimizing Z. 
这一步是计算反转 Z 的概率，如果反转某个 Zj 的收益更高，那么就反转 Zj；
// 计算的公式在原文中，可以理解但是不知道怎么才能这样设计；

Optimizing Y.
这一步将z 看作常数，因而 mv 的开销看作是固定的；
要求 收益的总和 Yij * B 最大化
可看作 ILP 问题，可以使用现有的 ILP solver 来解决 // 这是什么东西？

B. RL based Method
为什么上述的解法不能很好收敛？// 解释看不懂捏QAQ

将问题解释为 马尔科夫决策问题

1）Markov Decision Process
总之把上面迭代的算法模型化为马尔科夫决策过程，然后使用强化学习来解决；

// 也就是说前一个解法其实没什么必要；

强化学习选择视图：策略是要选择视图 Z 集合，动作是选择一个视图 Zj，reward 是计算得到的 utility 变更；

2）RL-Based Method
Q-learning
value-based RL algorithm； // 如果写的话，需要一些展开的介绍 // 就随口一提

// Q learning 原理是什么？ Q table？ 里面包含一堆 Q values，然后怎么更新？

state 与候选subqueries之间是指数关系，所以搜索空间还是很大；

DQN
最后采用 Deep Q-learning Network 来预测； // DQN 怎么工作，还不太懂；

6 Experiments

数据集是 IMDB，workload 是 Join Order Benchmark（另外实验中手动添加了 query，使得 workload 出现更多重复计算的查询；（翻倍

另外的数据集来自 Ant-Financial https://www.antﬁn.com/


数据集处理方面，统计了 投影的数量、表的数量、查询数量和子查询数量
随后用 EQUITAS\[45] 的方法检测公共子查询，统计 equivalent pair 数量；
这一步之后将子查询划分为不同的 group，每个 group 中选出开销最低的子查询作为候选子查询 #candidate subqueries |Z|；
把 workload 中能够利用这些子查询视图的 查询 称为 |Q|； // 这部分在整个 workload 中占比较少； 在真实数据集中占比约 10%，JOB 可能因为手动构造的缘故几乎全部相关；


Baseline：
1）Cost Estimation
评估代价预测模型，与其他现有工作进行对比:
a）传统评估方法： \[20] ；
使用 Postgres 9.1 优化器来评估 JOB，和 MaxCompute 评估 WK1 & 2;
然后使用 \[36] 的深度学习方法来对比；
b）LR：使用 linear function 来计算估计cost
c）Gradient Boosted Macine：XGBoost\[5]
d）修改了 W-D 模型中不同部分，来探究它们对整体模型性能的影响；


2）View Selection
与迭代方法 BigSub \[20] 和四种贪心方法进行了对比，这些贪心方法是：TopkFreq, TopkOver, TopkBen, TopkNorm  \[10] // TODO：这些贪心都是啥，看看论文

a）BigSub：将查询标为二分图，然后将视图选择问题看作 标记 V/E 的问题？ // 啥意思？

b）贪心：将候选子查询根据不同策略进行排序，选出 top-k 子查询进行物化；

TopkFreq：频率最高的 k 个
TopkOver：物化 overhead 最低的 k 个
TopkBen：增益最高的 k 个
TopkNorm：utility 和 oveahead 比值最高的 k 个 // 这是旧论文常见的；

Ealuation metrics：
MAE（Mean Absolute Error）
MAPE（Mean Absolute Percent Error）

workload 按照 7：1：2 的比例划分为 training、validation 和 test，使用 Adam 优化；

Parameter Settings：跳过没看

B. Comparison with Baselines
1）Cost Estimation：
对于 JOB，重写了 queries 然后实际执行得到 actual computation cost；
但对于 WK1 & 2 两个较大的 workload 无法这么做，这里是通过 RealOpt 方法获得渐进结果作为 ground truth； 执行 Q 和 S，然后用 cost（Q）- Cost（S） 当作视图 Vs 的 ground truth；

a）Optimizer 的估计准确率最低；（但是我之前看到的文章中，用合适的 sample 误差会在 5% 以内？

b）基于神经网络的方法表现最好，

c）但在 JOB 上表现差于 WK1 & 2,可能小数据集导致了过拟合

d）W-D 模型估计开销表现更好，一方面是估计代价时考虑了更多信息，另一方面抽取了更多特征；

e）W-D 的变种实现表明：

f）对于 JOB 中的 10 table join 和 一个视图的 3 table join，W-D 模型表现不好；

2）View Selection
对于贪心的 TopK，实验中将 K 从 0 ～ 候选子查询 进行调整，在 K 逐渐增大到某一范围时，utility 上升到峰值；而后随着 k 增大而出现一定程度下降；

分析原因是随着选择视图（子查询）数目的增多，物化的开销逐渐超过 视图减少重复计算带来的收益；


实验中的算法分为两类：1）贪心；2）迭代遍历（暴力枚举？
后者在各个数据集上的表现均优于贪心

分析原因：1）贪心无法保证最优，比较难达到 max utility；2）iteration based method 会探查更多的 candidate，有更大概率找到实际最优解；

iteration based method 中的 RLView 表现最好；


Ealuating the convergence
IterView (是中间步骤还是啥来着？)
不太能收敛；

RLView 能稳定收敛，

比较来说， IterView 只考虑局部最优结果，因此会在不同的局部最优解之间波动；
而 RLView 中的 DQN 方法能存储过去结果，因而不会出现

C. End to End Experiment
综合采用不同的 cost estimation 和 view selection 组合，来评估各个模型的表现
分别是 Optimizer + BigSub, Optimizer + RLView, W-D + BigSub, W-D + RLView，然后从 WK1 & 2 数据集中精简出一部分查询用于该实验；

这部分感觉，，，差强人意；

7 Related Work

// 简要概括一下，不过这篇文章的 related work 写得确实不错值得学习；

Cost Estimation
如上文所述，传统的代价估计主要依赖于优化器，而优化器是根据 cardinality？ 这是什么 或者 tuple 数量来进行估计的； // 给出不少参考文献；\[14,18,19,35,32] // 这些是传统方法；

然后出现了深度学习进行代价估计的方法，如 CNN，RNN，神经网络等用于 交通时间预测，查询开销预测 // 这部分给出的参考文献 \[24,36,44]

Subquery reusing
文献 \[28] 是不是这方面综述？ // 焯，2012年的 view selection survey

一些关于复用 common subqueries 的工作 \[3,20,31,34]；

一些数据仓库内的 view selection 工作 \[11,12]

SQL equivalence

