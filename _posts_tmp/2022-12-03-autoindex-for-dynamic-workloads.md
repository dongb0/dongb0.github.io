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

// Q1：如何评估索引更新/删除的开销？


abstract

许多工作专注于添加新 index 来提升读取性能，但是存在局限：1）现实场景中查询数量巨大，难以在给定资源限制内找到最优索引；2）没有考虑 index 的更新和维护开销，这可能会严重影响写入性能

因此提出一种面对动态工作负载的 增量索引管理系统 AUTOINDEX；

为了支持动态索引管理，将查询映射到 index templates 中，并且能通过模板产生可靠的候选索引
通过蒙特卡洛树搜索（Monte Carlo Tree Search）来增量添加或者删除索引，来保证系统的吞吐率。

同样，对于索引的 benefit 估计也通过深度学习模型来完成；// 都喜欢搞这种？

在 openGuass 上实现，表现超越现有方法；

1 Introduction
索引无非是以存储空间和写入时的维护开销作为代价，换取读操作的更优性能；

index management system 的职责：
1）保证索引空间占用在给定限制之内
2）workload 性能最优


Example1：如果由 DBA 手动创建索引，如文中展示的一个 银行取款业务场景，2.2 M 查询，DBA 经过精心设计创建 263 个索引，但是 由 AUTOINDEX 管理后，可以仅仅使用 44 个索引就能达到原有的查询性能，节省了 70% 的存储空间。

Example2：举了疫情数据库的例子，比如记录着每个被检测人员的体温、所属于的社区，在疫情发生之前，查询是随机的，这是可以对访问频率较高的体温和社区列建立索引；当疫情发生之后，因为 insert 操作增多，假设此时体温列的维护开销，大于它节省的查询开销，那么 index management system 可以考虑把该索引删掉；当疫情得到控制，workload 变为 update 居多，此时可以建立 （id，体温）的复合索引加快体温更新操作；

Key Observation：
1）workload 对于索引可能有不同的要求（在不同阶段？），所以需要能够动态调整索引来保证性能
2）索引的收益（benefit）由读查询减少的开销和写查询承担的更新开销共同决定
3）不同的数据列之间也许存在关联，一个 multi-column 的索引带来的增益可能大于多个 single-column 索引；

Limitation：大部分现有的索引管理系统，都依赖于优化器来估计索引的增益，并且贪心的选择带来增益最大的索引，这样的方式存在三点不足：
1）大部分算法关注点是在单个 query 层面，比如只要索引对至少一个 query 有益，就将其加入系统，这样忽略了索引之前、查询之间的关联
2）候选索引数量非常多，使用贪心算法可能只能得到次优解
3）基于优化器来估计索引收益的方式，忽略了索引的维护开销

同时也有一些基于机器学习的索引管理系统，

文献\[21,25] 使用 RL 来估算索引增益；
文献\[8] 使用深度学习模型估计索引开销；

这些方法的局限性：
1）DRL-based index selection：需要很长的训练时间，从数天到数周不等，无法应用在动态工作负载场景中，且 DRL 方法对于索引的删除通常不能支持；
2）DL-based estimation：对于单个 query 效果不错，但是对于 workload 而言不够理想；可能出现 performance regression，如某些查询只出现一次，或者推荐的索引经常被 更新；这些方法面向 analytic 查询，不适用于估计 索引的 更新开销；

Challenges：
1）如何高效的捕获 workload 对索引的要求
2）如何高效更新索引
3）如何估计读写查询的增益

根据 incoming query 和历史查询，进行索引的更新来保证吞吐

AUTOINDEX 可以
1）监控 performance 检测索引导致的性能下降
2）使用增量的索引更新方法，从 template 中获取可靠的候选索引（for C1），使用蒙特卡洛树搜索来选择高收益的索引（for C2），并且在 蒙特卡洛树 搜索过程中，使用了一个深度回归模型来估计索引的增益（for C3）

Contributions:

2 Preliminaries

A. Index and benefits
定义单个和多个索引的增益

单个索引增益：对于 workload W 和索引 I，I 的增益 B(I) 定义为 B(I) = 求和(cost(W) - cost(W.I))，即不使用索引 I 时 workload 的总开销 与 使用索引 I 后 总开销的差值

多个索引增益：多索引的增益计算公式类似，但索引之间可能有关联 // 如何体现？

B. Index Management
静态工作负载的索引管理:

定义：给定一个候选工作负载 W，和存储空间限制 B，从候选索引集合中选出一个子集 I，使得：1）I 的大小在 B 的限制之内；2）负载的性能最优

动态工作负载的索引管理：
实际场景中，往往不能预先估算得到整个工作负载，因此需要动态的对索引进行调整；

incremental index management (IIM)

定义：
能根据新的查询更新索引集合 I，得到 I‘，使得
1)I’依然满足空间限制 B；
2）I‘带来的增益大于 I；

3 System Overview

使用蒙特卡洛树的动机
索引更新是 IIM 系统的关键部分，IIM 需要：1）移除低增益的索引；2）添加可靠的索引
同时还有有能力对索引之间的关联进行分析（否则贪心的选择可能会导致陷入次优解

// Q：那么 MCTS 如何能够达到这些要求？
// in sec 4？ 

基础的 MCTS 仍然存在问题：
1）面对 large search space 依然运行很慢
2）动态 workload 可能导致树节点的 value 失效
3）需要可靠的 cost estimator 来对索引增益进行估算

workflow：
对于数据库新的 workload，进行性能检测，如果发现 performance regression，则生成候选索引，使用 MCTS 在现有索引和候选索引中进行搜索，找到最优的推荐索引集合；按照该集合更新现有索引


Index Diagnosis：
检测系统指标，如果发现 异常，则调用 index analysis 模块分析是否需要更新索引；
比如查看 没有创建的增益索引、使用次数少的索引、负优化的索引比例，如果达到某个阈值则发起一次索引推荐任务（调用 Index recommendation

Index Management：
索引推荐模块输入：workload 和 索引统计数据，输出：推荐索引集合
这部分包括三个模块：SQL2Template、Candidate index generation、Index Selection

因为 workload 包含的查询可能比较多，所以通过 SQL2Template 将查询映射到一定数目的 query template 中（能够表征最常访问的 query 模式

对于每个 template，通过 Candidate index generation 模块对每个 子句的谓词进行分析，并且基于谓词中包含的 column 生成候选索引；

//////////
按照这里的例子，生成的候选规则似乎很简单？比如出现谓词 a = ? and b < ? 就生成一个候选索引 (a,b)

最后使用 Index Selection 从候选索引中得到推荐的索引；

这里还维护了现有索引的决策树
///// 用来干嘛？

对数据分区的场景支持 index type selection
// ？这是啥
e.g. global 索引查询速度快，但消耗更多空间；local 索引查询慢一些，但空间小

Index Estimator：
现有数据库通常不能估计索引的维护开销；因此这里提出了一个 deep regression model（通过历史数据训练得到），输入 workload features 和 index，能够估计 workload 的执行开销

4 Incremental Index Management

传统的索引管理方法存在的问题包括：
1）如何从真实的数据集中高效的获取候选索引；在包括数百万个查询的 workload 中，对每个查询逐个进行分析代价是很高的；
2）如何表示这样大的一个 index space，并且从中高效的搜索索引组合；现有方法往往是启发式的贪心搜索，这会忽略索引之间的关联性，忽略多个索引带来的共同增益，从而导致次优解；
3）面对动态 workload 如何调整（传统方法做不到，本身是面向静态的算法，除非将新的查询与原有查询合并为新的 workload，再重新跑一次整个算法 // 我能想到的

决策树 policy tree：
文中构建决策树来表示索引空间和支持增量索引管理；
root 表示初始索引集合，包含主键和一些其他的 distinct columns；其他节点表示可能的索引集合；使用决策树的好处是可以表示已经选择的索引集合（已经访问的节点）和新的索引组合（未访问的节点）

！！使用决策树可以比较好的解决上述传统方法的三个问题；
// 考虑一下是不是可以借鉴？

-------------------- 这部分在 view selection 里可能不太会用到

A. Template-based Candidate Index Generation
生成候选索引包含三个主要步骤：

Step 1: Reduce workload size by mapping queries into templates.
这里减少 candidate 的方法是把大量（可能重复的）查询映射到 template 中，每个 template 代表一类具有相似索引要求的 query；

对于每个 query，将它归到最相似的 template 中去 
// TODO：这里具体怎么做？ template 如何生成？怎么才能得到比较全面的 template？

如果没有 template 匹配，那么就把这个 query 记录为新的 template；
但是系统中使用的 template 数量设定了上限（比如对于 TPC-C 设为 5000）；到达上限后对于新 query 将会对 template 进行更新；
this trick is useful.

Step 2：Exploit candidate indexes based on the matched templates.
这一步将使用 step 1 中匹配到 query 的 template 来生成 candidate；
有两个问题：
1）index 可能来自不同的子句 

// 什么意思？

2）布尔 predicate 可能由多个 复合表达式构成，这些表达式有不同的等价形式，（如果不做处理可能会得到不同的 index 组合？
比如 (a AND b) OR (a AND c) 和 a AND (b OR c) 是等价的，但是按照表达式直接解析对应的索引的话，前者会得到 index(a, b), index(a, c)，而后者会得到 index(a, b, c)；如果 b OR c 对应的结果很多那么可能会得到较差的性能；

为了避免这一问题，可以将每个子句的提取出来单独生成 index（expression extraction)

Expression Extraction:
把 template 分为三类：filter predicate，join predicate，other expressions；
1）filter predicate 可以提升单表的查询，生成单列索引用于atomic predicate，多列索引用于 复合predicate，如多个 ANDs
2）join predicate 通过 building indexes for driven table ///
/// 啥意思？？
3）其他类型表达式比如 GROUP BY，ORDER，就直接对这些子句包含的 column 建立索引来提升性能

Index Generation：
对于每种不同的 template 分别生成 index：
1）filter predicate：
首先将 boolean predicate 重写为 Disjunctive Normal Form（DNF）
// 这是什么？
对于 atomic/composite predicate，如果 selectivity 大于某个阈值（比如设为1/3，可能是筛选的数据比例），才生成对应 index
2）Join Predicate：
将两个表的 atomic join 提取出来生成对应列的索引；
// 是所有的 two table join 都这样吗，那多个列上进行的 join 是否不属于 atomic？
3）other expression：对于 GROUP BY/ORDER ，如果表达式生效了（比如group by 中的列对应行数的确不唯一），那么将其包含的列生成候选索引；

Step 3：Check and remove redundant indexes. 删除重复索引，根据 left most 原则合并索引，如 index(a, b), index(a)，只保留 index(a, b)，并删除已经存在的索引；
剩余部分就是候选索引；

B. MCTS-based index update
经过上一步得到的索引数量已经大幅减少了，接下来使用 MCTS 在决策树中，根据候选索引集合来更新现有索引。决策树中每个节点都代表一组现有索引或候选索引的组合；所以 index update 就是扩展决策树，来寻找能带来更高性能提升的 树节点；

但此时 index space 还是很大，为了能更高效的寻找最优节点，这里基于 MCTS 提出了一种新的搜索策略？

MCTS 的核心思想是 平衡  exploitation/high benefit 和 exploration/low frequncy 来避免陷入次优解；

这里先给出决策树中节点的 benefit 定义：
Node Utility:

如果一个节点位于 root 到 最优节点的路径上，那么它会有更高的 utility；但是因为我们不能预先知道最优节点，所以很难判断一个节点是否位于最优路径；

这里考虑两个因素来计算 node utility：

1）Node Benefits(B_vi)：在给定的决策树中，定义一个节点 vi 的 benefit 为 vi 或 vi的后继（descendent nodes）最高的节省开销

cost(W) - cost(W.I) leaf node or
max(cost(W) - cost(W.Ij)), vj 属于 Vj otherwise

因为一个节点的后继可能非常多，这里选择的是随机访问若干个节点的索引，直到达到存储限制（比如 5 个节点）；一般来说我们希望访问 benefit 最高的节点，但是如果一直这么做就有可能错过最优解；因此同样还需要考虑平衡节点的 benefit 和 frequency；

2) Access Frequency F(vi):表示选择新的 child node 时访问节点 vi 的频率；除了考虑 benefit，我们还需要尝试探索一些不经常被访问的节点，来避免陷入局部最优；

因此综合定义 upper confidence bound 为 // 公式这里省略一下；
形式上很类似 MCTS 的那个（本来也是一个，改了具体参数而已）

C. Incremetal Index Update
更新过程分为三步：
Step 1：Node Selection and Expansion.
选择 utility 最大的节点（ 最有可能得到最优解）
i）如果节点没有被扩展，则枚举每个候选索引 Ii，并且在存储空间限制内添加新节点 v + Ii
ii）如果节点已经被扩展了，那么选择 utility 最大的 descendant node

Step 2：Node Utility Computation. 对于选择的节点 v，计算它对应的 benefit；并且随机访问 v 的 K 个 descendants 并取其中最高的 cost reduction 作为 v 的 benefit；

这里使用了 sec 5 介绍的代价估计模型，来获取更高准确的估计结果（同时考虑读和写查询；

Step 3：Utility Update
节点 v 的 utility 更新之后，它的祖先节点对应的 utility 也有可能需要更新，因此要将更新向上传播；

迭代重复上述步骤，直到达到最大迭代次数或者性能趋于最大

C. Incremental Template Update
因为我们依赖 template 来生成候选索引，所以当 workload 变化时 template 的更新也非常重要；

但是预测 workload 并不容易，这里总结归纳一些实践经验：
首先，对于大部分 workload 总是存在一些 query 反复出现的，因此基于历史数据，我们能够预测相似的 template 可能再次出现；可以采用 LRU 替换策略；

其次，当 workload 可能大幅变化时，可能会出现一些其他的规律，如大部分 historical template 的更新频率低于某个阈值，此时我们考虑大幅减小 template 的访问频率阈值，移除低频率的 tempalte，并替换成最近出现的 query；

5 Index Benefit Estimation
// 这部分简单看看，做视图的不一定能用

重申传统方法基于优化器进行代价估计可能发生重大偏差；基于深度学习的预测方法可以得到更高的准确率，但主要只用于分析查询（analytical queries）

估计开销时，要解决三个问题：
1）实际创建索引的时间空间代价，这里通过 hypothesis index 模拟的方式来解决；
2）PosgreSQL 的 cost estimator 功能比较强大（估计值比较准确？），但无法估计索引的更新开销
3）传统代价模型基于有权重的代价特征，这些权重是固定的（静态的），这就可能造成误差；
// cost estimator 到底是怎么样估算开销的?

因此这里采用历史索引数据来训练深度回归模型，能够动态学习代价特征的权重，提供更高的估算精确度。

首先定义
Execution Cost:
一般而言，查询 q 的执行开销是根据 IO 和 CPU 消耗来计算得到；如果 q 是写查询，则还要考虑索引更新的开销；因此将查询开销定义为

cost(q) = Model(C^data, C^io, C^cpu)
C^data 是数据的处理开销 // 定义有些模糊？什么叫处理开销？
后两个是索引更新的 io 和 cpu 开销

A. Computing Cost Feature
基于内核的 statistic，可以比较方便的估算 IO 和 CPU 开销；

IO Cost
可以根据访问 page 的数量估计 IO Cost，定义为 const * |pages|

CPU Cost
使用访问 tuple 的数量来估算 CPU Cost，C^cpu 定义为 T_start + T_running
即 寻找对应 index tuple 的开销 + 插入或更新 tuple 的开销
T_start = {Log(N) + ( H + 1) * 50 } * cup_operator_cost // 这个怎么估算?
T_running = N_insert * cpu_insert_tuple_cost(const param)

B. Deep regression for Cost Model

回归模型（深度学习），加入了更新索引的度量
// 原本的代价模型是什么样子的？反正就让它自己学去

// TODO:有没有其他考虑索引维护开销的方法？

// 整体看下来，似乎没看到是通过什么方式来实现 根据动态 workload 进行索引调整的？
好像就是检测到查询性能出现下降后，发起一次索引分析任务？这思路好像也非常简单？
// 整体而言似乎与那些静态的方法也并没有什么不同啊？

6 Experiments
