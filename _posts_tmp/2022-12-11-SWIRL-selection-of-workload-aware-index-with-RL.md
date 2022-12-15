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

// 简单过一下这篇论文，只看感兴趣的部分；

abstract

1）具体使用了什么RL方法？
2）如何泛化到新的 workload
3）如何通过invalid action masking将模型扩展到支持数千index
// 4）数据集TPC-C，TPC-H，JOB // 这部分就不看了


1 Introduction

RL based model 在训练之后，利用从历史数据中学习得到的知识，能够“立刻”推荐出适合的索引，而不是像其他方法一样要通过 generation、考虑 index interaction 等来进行选择；

// 那么如何做到的？

contribution：
1）RL-based approach，性能相当，但速度更快
2）使用了一个 workload model 来生成新的 queries，并且使用 invalid action masking 的方式减少训练次数
3）在 PG 上实现，并提供 RL-based method 的 survey；
4）公开源码

2 Background

2.1
2.2
2.3 Reinforcement Learning
// 描述RL的过程，agent，action，reward



3 Related Work

3.1 Existing Index Selection Algorithm

reduce（从一整个 configuration 往下删 index 的方法）通常运行时间比较长；

文中选择对比三个 additive 算法

3.2 RL-based Index Selection Approaches
对比了一些现有 RL 算法；

3.3 Research Gap: Requirements for Competetive RL-based Index Selection

提出了一大队 requirements（6条），论述现有方法无法满足这些要求；

4 SWIRL: Selection of workload-aware indexes using RL

4.1 Overview

框架分为三部分：
1）pretrain：生成训练数据集和测试数据集，决定候选索引，生成 workload representation model（这个model到底用来干吗？
2）training：agent 学习哪些候选索引是有用的，以及索引之间的interaction
3）application：agent将训练的模型用于 workload 选择索引

2）和3）构成马尔科夫决策过程；

preprocessing
1）DBA 指定 一组representative query；
2）根据schema 和 query 生成 candidate；但也会进行筛选（//根据什么筛选？）这里生成candidate的策略大概是：选择所有语义相关的（syntatically relevant）索引（包括它们的permutation）；对于比较小的 table（n<10000）不生成 candidate；
3）根据representative query 生成 random workload；生成方法是：选出一个 query 的子集，然后重复每个查询不同的次数；也就是 为了让最后的 workload 包含不同频率的查询； 然后将 workload 分为训练集和测试集；
4）将query 变为 numeric representation，喂给模型；细节见 sec 4.2.2

Training
RL 在索引选择中的主要组件是：agent和environment；
stateless agent每次行动表示选择（创建）一个索引；
stateful environment 则将 agent 的行动映射到数据库中，评估action的reward（对应数据库中索引带来的收益）
--接着preprocessing的序号
5）environment获取一个 workload
6）训练过程中，向优化器询问某个query使用某个configuration的查询计划和代价
7）限制action space，产生符合当前 environment状态的action；这是能够快速收敛的关键，也是跟现有RL方法的主要区别
8）将environment state转化为 numeric feature，传给 agent；
9）10）11）agent 根据 feature 和 action space 选择行动，
12）environment 通过 what-if calls 完成 action（变更状态，

然后重复6）～12）直到没有有效的action为止；
// 这里为什么 2.3 提到一个 ANN ？

Application
训练完成后，实际部署的流程是：
Tranning 流程中的 5）将会获取一个真实的workload，然后agent会根据训练好的 ANN 来选取 预估reward 最高的行动 a_t（在状态 s_t）;重复这一过程直到达到空间上限；

// 但是没有解释为什么可以用于 unseen workload（可以用，但是此时的性能呢？

模型为什么快？
1）模型部署之后不再需要 what if calls
2）训练好的ANN模型只需要进行简单的evaluation，（传统方法要怎么做？

4.2 Model
介绍如何将 index selection 模型化来使用RL解决；
展示如何将环境信息（指workload和configuration）转化为向量表示，以及如何表示workload，以及如何将config的变化模型化为action，以及agent的reward；
// 确实比较好奇如何把 query 转为 vector 表示的


4.2.1 State Representation
（简化版本的representation示例
将workload抽取28个特征；

状态表征大概分为三部分：
1）workload
2）meta info（budget
3）current index ocnfiguration


workload representation
我们要求 representation 能够反映 query 的不同类别、不同频率；

对于一个包含N类查询、representation width 为 R 的 workload，对应的 vector representation 将会是：
1）N 个长度为 R 的向量，每个向量代表一类查询的编码
2）一个长度为 N 的向量，记录每类查询的频率；
3）一个长度为 N 的向量，记录每类查询在使用当前 configuration 对应的开销估计值；
其中 1）对于查询的编码表示方式是本文的创新点

网络的规模 N 是固定的，考虑实际 workload 导致 N 不同的情况（比方说实际为 N'：
如果 N' < N，则可以使用 padding
如果 N' > N，那么依然可以使用 N 大小的网络，只取其中最重要的 N 部分；
这也有些相关的工作：workload compression；文献\[11,16]
也有 query clustring 文献\[37]
选择 N 的大小应该也是调参的一部分；

The Meta Infomation
包含几个标量：
1）指定的存储上限
2）当前存储使用情况
3）workload 未使用index的初始cost
4）优化器预估的当前cost

The Current Index configuration

文献\[21,50]有些研究，
wide index 在实际系统中很常用？
// 但是跟我视图有什么关系
总之这里没有考虑过多限制（减少）candidate数量

// 但这样的话，算法如何进行搜索？
虽然没有减少candidate数量，但是通过对每个 attribute 单独编码的方式，来减小了特征维度；
具体做法是：将每个value按照attribute的位置增加了 1/p，其中p是attribute在索引中的位置；

这样编码是希望能够得到（记录）某个attribute被index使用的信息；似乎agent也能够学习出这一模式，并且足够应用 masking 方法；

Concatenation and normalization
进行训练前要对vector进行拼接
// 按照什么方式？
以及归一化

归一化是为了提升网络的学习表现；activation function 本来可能出现大规模输入下梯度消失的问题？// 就是通过这种方式来解决的；

number pf features
大概的估算就是上面 fig 4 例子中的 feature 数量
F = N * R + N + N + MI + K（被至少1个query包含的attribute
对于 TPC-DS，上述计算的特征数量约为1750

4.2.2 Workload Modeling and Query Representation 

文章的一个主要贡献：能够处理（应对）不在训练 workload 中的查询模板；
（但这些查询还是需要与现有查询有相似，而不是完全不同；


通过反复调用 what-if call 来生成不同索引 configuration 下的执行计划;
// 理论来说也可以直接使用SQL来生成 query representation，但文章认为SQL的执行计划包含着更多信息；

然后将 operator 转化为 text representation；
比如对于 TableA.col4 上的索引，可能表示为 IdxScan_TableA_Col4_Pred< ，然后存储在 opertor dictionary中；然后使用 BOO（Bag of Opertor, bag of word 模型来处理）
文献\[10,25]

Dimentionality Reduction
如果对BOO不做处理直接使用，可能会导致维度爆炸；
所以这里采用了Latent Semantic Indexing 模型来减少特征数量 
// 文献\[17] 
也有其他方法，如 random projection \[3] and principal component analysis

设置query representation特征的数量（即上述的 width R），是在表征信息的丰富程度和模型大小、训练时间之间的 trade off；

文中在数据集实验后发现 R 取 50（对于这些数据集比较合适

Alternative workload modeling approaches
// 对比其他研究的 modeling 方法，暂时不看

Simplification and extensions
不考虑unseen query会简化模型（那么现在复杂在哪里？

如果相同的query 但 cost 不同，这一差别会反映在 query cost vector 中，

另外 BOO 模型对于语义相同但结构不同的 plan 可能无法区分，这可以通过将 preceding operator 一同编码在 representation 中来避免；但目前似乎没遇到这问题导致的问题，所以文中也没实现；// 那你还写？

4.2.3 Action
action space（很关键，将决定 agent 能创建哪些index；
文中模型的 action space 是离散的，每个action代表一个index candidate；
而这些 candidate 在预处理阶段生成（sec 4.1），这可能导致数千个 multi-attribute index candidate

必须要谨慎设计action space，
1）首先复杂的actoin space会使得问题变复杂；而提前筛选candidate虽然能简化，但是又可能影响性能
2）某些 action 在特定环境下可能是无效的；比如重复创建已有索引、创建索引将超过存储限制等；通常 RL 中是通过给这些违反的action设定一个比较大的负奖赏，让agent能学习到这是无效的行动；但需要agent自行学习也意味着训练时间需要变长（在Q-learning的demo里已经看到了

因此这里使用 invalid action masking 的方法来暂时将某些action标记为 invalid；这需要每次行动前对valid action进行更新；
有四类情况是可以用（需要用）masking的：
1）索引与当前workload语义无关
2）超出budget
3）索引已经存在
4）invalid precondition 比如创建了single-attribute index A 之后，所有double- multiple- 并且以 A 作为first attribute的索引被标记为 invalid；
这是来自 文献\[12, p151] 和文献 \[50] 的extend算法的思想

也可以将DBA希望保留的索引标记，避免算法的对其进行修改；（比如从集合中去掉

4.2.4 Reward
将索引带来的增益与消耗存储空间的比值，作为reward；
并且因为使用了 masking，所以没有必要对 invalid action 做出惩罚；

4.2.5 Miscellaneous
一些训练技巧：如果训练过程中性能没有得到提升了，则保存下当前模型的状态；
另外缓存了一些开销比较大的cost request；（

5 Implementation




