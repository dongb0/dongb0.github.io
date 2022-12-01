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

Materialized view selection


1 Introduction
视图选择问题常见于查询优化，数据仓库，分布式data placement 等场景，综述\[21]关注使用 MV 重写查询的方法，综述\[32]关注 web database 的物化方法，\[13]关注数据仓库的物化视图选择方法，但这些综述并没有对视图选择的方法进行分类，对比其优缺点。而本文即将做这项工作。

主要关注关系型数据库和数据仓库（包括分布式环境下）的视图选择算法。
首先定义视图选择问题，然后对视图选择方法进行分类。

sec 2 问题定义
sec 3 讨论视图选择的 dimension
sec 4 survey（但是都包括哪些内容呢？
sec 5 结论 & open issue

2 Problem Spectification

2.1
View：在基础表/其他视图上通过查询定义得到的关系（表）
Materialized View：将内容持久化的视图
Workload：工作负载是一组查询序列 Q={Q1,...,Qq}，每个查询 Qi 包含一个非负的权重 fQi 表示该查询出现的频率。分布式场景下这些查询将在不同节点上执行，每个节点将会有不同的工作负载。
View Selection：给定数据库模式和工作负载，从中选出一组适合的视图来提高查询执行效率，称之为视图选择。
View Maintenance：当基础表改变时，与其相关的物化试图也需要更新来提供最新的查询结果，更新物化视图的这一过程称为物化视图维护。有不同的维护方式（deferred/immediate）和策略（incremental/remaiterialization） // TODO： 四篇文献\[17,18,38,58]

2.2 Problem Definition
需要权衡的收益：查询处理时间、视图维护时间、存储空间。选择要维护什么样的视图来平衡三种开销，得到最理想的结果，就是我们需要解决的视图选择问题。也是 NP-complete 问题。在分布式环境下，视图选择问题变得更加复杂，（具体看文献\[16]，可能包括数据粒度，数据拷贝数量，等）

Problem Formulation：
给定模式 R={R1,...,Rr} 和在R上的工作负载 Q={Q1,...,Qq}，要选择适合的一组物化视图 M = {V1,...,Vm}，使得工作负载Q在给定资源下（有限空间、视图维护开销）能够获得最低的查询代价。

分布式场景下的视图选择则需要在一组节点 N = {N1,...Nn} 中执行查询，此时的资源限制可能变成每个节点能使用的存储空间、通信开销、视图维护代价。

2.3 Cost Model 代价模型

QueryCost= 

每个查询的频率 * Qc(Qi, M) 使用物化视图执行查询 Qi 的代价

还要考虑物化视图的维护开销：
ViewMaintenanceCost = 

视图 Vi 的更新频率 * 给定物化视图集合 M 来维护视图 Vi 的代价

在分布式环境下，还需要考虑通信开销（传输延迟、传输数据时间开销）：
给定查询 Qi 在节点 Nj 上执行，需要使用视图 Vk 来处理查询 Qi，那么当 Vk 存储在 Nj 上时通信开销为 0；否则通信开销定义为 Cni,nj * size(Vk) ：节点 Ni 到 Nj 的传输开销 * 视图 Vk 的大小

可以合并为大括号的公式： {

2.4 Static View Selection vs. Dynamic View Selection

静态视图选择是基于给定的工作负载，但动态视图选择则是根据不断到来的查询序列来选择视图，所以动态选择的算法需要能够增量更新调节。

在动态的系统中(文献\[4,5,29])，选择的物化视图集合是随时间改变的，当工作负载变更时会选择能提供更多收益的视图来更新现有视图集合。 // TODO：更改 MV 的开销如何衡量取舍？

文献\[57]选择维护视图中最常访问的 tuple 来减少 MV 的维护开销和存储空间。

文献 4 是踏马法语还是什么语言的，
文献 5 是 P2P 的 动态选择
文献 29 是 Dynamat，大概是缓存了？ // 怎么讲不清楚？
文献 57 是 SQL Server 上的那个，只缓存视图中一部分经常访问的数据，并且缓存的数据可以根据设定的 LFU/Q2 等替换策略更换；

dynamic view selection -> view caching

一种方法是缓存 chunk of view，而不是整个视图；文献xx设计了一个新的组件 chunk file 来计算（存放？那些不在缓存中的 chunk。

文献 \[28] 搞了分布式数据库的 cahching view selection？
// 这个文献有四十页，2000年的，就没看

-------------------------------------------------
但是本文只探讨静态视图选择算法；（因为大部分算法都是静态的）
-------------------------------------------------

3 View Selection Dimensions

文章提出 1）frameworks 2）资源限制

3.1 Frameworks
一般而言，视图选择算法有两个主要步骤：1）确定候选视图（来执行物化过程），有DAG方法、对workload的语义分析、查询重写等；2）根据选择的视图，在给定资源限制下进行物化

// 不过这具体是如何构建出来这样的 DAG 图的呢？》

3.1.1 Multiquery DAG
大部分提出的索引选择算法通过查询执行计划来运行。可以通过查询优化方法获取查询计划，通过检测公共子表达式（sub-expression）和捕获其中的依赖，（可能是主要的研究点？main insterest）

这些特性可以用于 share update and storage space；依赖关系可以通过有向无环图来表示（DAG），但这需要调用代价昂贵的 optimizer calls；

文献中常见的 DAG 有：
AND/OR View Graph：
\[40] 中提到将所有执行计划的并集构成 AND-OR View Graph。。。是啥啊啊啊？
由两种节点构成：
Operation Node：代表一个代数表达式？如 Select-Project-Join
Equivalence Node：代表一系列相等的逻辑表达式（如产生相同结果的表达式）
// 图

Multi-View Processing Plan
\[52] 提出的 MVPP 同样是有有向无环图，根节点是 qeury，也节点时基础表和其他中间节点（视图），如 selection projection join or aggregation View；


// 看不懂


Data Cube Lattice：
\[22] 提出一个多维度的数据模型？用于 OLAP的 // 那算了不细看了

Data Cube Lattice ，也是 DAG，节点表示 query/view（按照group by的属性分类），边代表视图之间的可达性（derivability），如 V1 有一条到 V2 的边，表示能通过 V1 group by 的属性计算得到 V2；

这种表示方法的好处在于 query 可以通过其他 query 表示？
\[3,53] 扩展了该工作；


3.1.2 Query Rewriting
找到 mv 集合之后，还会找到基于这些 mv 重写的查询；采用这种方法的输入不是 DAG 而是查询定义（query definition），视图选择问题转为 state search problem，
/// 看不懂，说了啥啥啥啥啥？

3.1.3 Syntactical Analysis of the Workload
也有基于对 workload 句法分析的视图选择；通过对 workload 分析来选择一部分 能够通过物化来显著减少代价的 relation ；不过 search space 可能会很大。

3.2 Resource Constraints

3.2.1 Unbunded
资源无限（存储空间，CPU），因此算法可以选择能够让查询处理开销最小化的视图组合。

最小化cost 查询的代价 + 维护视图的代价 求和

存在两个问题：1）选择的视图可能占用过多存储空间；2）维护视图的开销可能超过使用视图带来的增益。

3.2.2 Space Constrained
在有限存储空间下，我们定义 view benefit 为通过使用该视图，能够给查询带来的代价减少量。基于此我们还可以定义 per-unit benefit，即视图增益与视图所占用的存储空间的比值。

在文献\[19] 中说明， per-unint benefit 只会随着选择视图的增多而单调递减（// TODO：有待确认

代价公式添加一个限制，视图 Vi 占用空间总和不得超过限制 S；

3.2.3 Maintenance Cost Constrained

视图的维护代价可能因选择其他视图而降低？（不单调

限制维护视图的代价不超过某个设定的 


4 Review of View Selection Methods
从以下维度对视图选择算法进行分类：1）考虑资源限制的，2）framework

4.1 Deterministic Algorithoms Based Methods
\[41] 1982 的文献第一次提出一种针对物化视图索引的 solution，基于 A* 算法。

\[31,39] 提出一种穷举算法，但不能在 resonable 的时间内得到结果（所以算了不要看）

\[22] 分析了用于 OLAP 查询场景的视图选择，使用多项式时间的贪心算法来选择能够最小化查询处理时间的物化视图集合，但没有考虑视图维护开销。

\[51] 的工作提出一种旨在适用于通用 SQL 查询（select/project/join/aggregation）的贪心算法，使总的查询处理时间和视图维护代价都最低；但他高估了维护视图的代价，同时没有对资源的使用进行限制；

\[19] 提出一种在数据仓库中部署的 几乎最优的指数&&贪心算法？for the case of AND-OR view 和多项式时间的贪心算法用于 AND 视图， \[20] follow 了这项工作，增加了对维护视图开销的限制；

\[42] greedy heuristic 用于多查询优化？ conjunction AND-OR
\[36] 扩展了该工作，添加了在维护视图开销限制下的优化

\[2] 基于 query tree ，空间限制，2 phase resolve：
1）预选一部分视图来进行局部优化（同时避免增加过多的维护开销）
2）计算（query graph）每个层级的开销，选择查询开销和视图维护开销最小的视图（具体怎么做？）

under the condition that the input queries can be answered using exclusively the materialized views. ？？？

穷举算法选 mv ？ \[47]，\[45,46,48]在空间占用的限制下解决索引选择问题，但是依然指数时间复杂度；
A survey of work on
answering queries using views can be found in \[21]


\[1] 对workload的句法分析，来选择需要物化的视图&索引。主要分为三步：
1）分析workload，选择查询过程中主要开销的 relation 子集
2）分析语义相关的视图和索引（是否需要物化
3）运行贪心枚举算法，选择需要物化的索引和视图 // 我想知道这个算法的具体实现方法啊?
缺点：没有考虑视图的维护开销

\[3,53] 在分布式的数据仓库场景下处理视图选择问题；提出了一种通过 data cube lattice 获取分布式语义的概念？
同时他们扩展了一种基于贪心思想的选择算法应用到分布式场景下；

缺点：使用的代价模型没有包括视图维护开销，同时也没有考虑网络传输的开销（但这在分布式环境下很重要），只是通过查询结果的大小来计算/估算传输开销。

上述的算法通过穷举或者启发式经验（如贪心），来选择视图，但是贪心算法可能会得到次优解，而不是全局最优。

为了解决这一问题，又发展出了 randomized algorithm 和 hybrid algorithm 和 constraint algorithm 等方法。

4.2 Randomized Algorithms Based Methods
遗传算法、模拟退火算法

借鉴自神经网络；

\[23,55] 在 MVPP 框架下应用一种遗传算法，
因为遗传算法的随机特性，有些 solution 会变成 infeasible？

选择某个视图来带的增益不单由该视图本身决定，也来自于其他被选择的视图；可以通过给 引入 一个 惩罚值 来解决这一问题（如何引入？

// 通过一个惩罚函数，在每次不满足维护代价限制时，减少 benefit？

要想遗传算法迅速收敛，可以根据经验设置一些初始的传播配置（完了开始往调参去了），比如倾向于选择被查询访问频率更高的视图； // 这些都可以不看，2012年的，机器学习在这方面的研究还不够深入

\[8] 基于对 workload 的句法分析来解决分布式的视图选择问题。算法分为三步：
1）扩展 \[1] 中描述的关系选择算法，
2）生成候选视图来进行物化
3）使用遗传算法来选择物化视图和网络节点来存储视图，使得查询处理和视图维护代价都最小化；

\[10,11,24] 采用模拟退火算法，

\[10] 选择物化视图的同时，考虑了查询开销和视图维护代价
\[24] 只在空间限制或最小化维护代价两个条件中的一个
randomized search 能够考虑空间和维护代价限制的情况，另外能够解决动态视图选择问题。


\[11] 提出了一种 并行模拟退火 算法 来解决大规模的查询场景下视图选择的问题。
但是这种方法没有对空间站荣和视图维护代价进行限制。

Solution Quality：也许可以通过算法提供的 solution 能够带来多少收益（如比起不使用该 solution 减少了多少 IO 或 CPU 使用等）


4.3 Hybrid Algorithms Based Methods
混合 deterministic 和 randomized 算法，期望能提供更好的性能和 solution quality。 使用 deterministric 算法来作为 solution 的初始配置，

\[56] 提出一种综合了贪心和 genetic 算法来解决 查询优化、选择最优全局计划、和为全局计划选择物化视图 三个问题。他们的实验结果表明，hybrid 方法能够比采用任意单个算法得到更好的性能。

缺点是他们的算法运行时间更长，复杂度高；


4.4 Constraint Programming Based Methods
Constraint programming is a descendant of declarative programming. ？

// TODO：跟 LP 区别是

\[35] 提出一种使用 Constraint Programming 的方法，通过将 视图选择问题 模型化为 满足 约束的 问题；作者实验证明，该方法由于其他的 randomized 方法，他们的工作 1）仅考虑维护代价 2）在维护代价和空间占用限制存在的场景下，同样可扩展 //？？？

5 Conclusion

文章分析了关系型数据库和数据仓库（包括分布式场景下)的视图选择算法，

state of the art 视图选择算法？ 有吗？哪些？

分布式数据库和数据仓库的视图选择算法研究较少，（p2p更少，论文中仅列举了一篇，但是我不管）


1）2012年SIGMOD综述中，大多数算法都是对静态的 workload 进行视图选择，即运行算法需要获取 workload 包含的所有查询；几乎没有能够进行动态视图选择的
2）在分布式场景下的选择算法研究较少（只有3篇文献提及，具体内容还没看；
3）论文也没有对各种算法的性能进行统一的对比，仅仅分析了算法特点和缺陷（可以考虑实现算法做一些对比实验，如果能力可及）


4）如何看该综述之后，view selection 算法还有没有新的工作？
