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


1）2012年SIGMOD综述中，大多数算法都是对静态的 workload 进行视图选择，即运行算法需要获取 workload 包含的所有查询；几乎没有能够进行动态视图选择的
2）在分布式场景下的选择算法研究较少（只有3篇文献提及，具体内容还没看； // 看了其中一篇，2003年的，主要是对一个贪心算法的扩展，不过里面提到 view selection 的子问题，在静态算法下还有个 Multi-view processing plan ？ 指出了对应文献，是否去看看；
3）论文也没有对各种算法的性能进行统一的对比，仅仅分析了算法特点和缺陷（可以考虑实现算法做一些对比实验，如果能力可及）
4）如何看该综述之后，view selection 算法还有没有新的工作？

// 物化视图常用于数据仓库冗余存储数据、提升 OLAP 能力


### 论文背景     

// 2012 年的综述，跟随它的被引用列表没怎么找到新的 视图选择算法 ； // 再核实一下
// 那之后才开始出现一些分布式的关系型数据库，比如 spanner，后来开源的 cockroach，tidb，ob 等；

### 实验思路

分布式的视图选择，目前看到的一个是贪心，同时也只考虑了空间限制； //再看一篇2009年的

// 能够借鉴索引选择里一些比较新的算法，搬到分布式场景下，实验对比其他贪心算法和 ramdonmized 的算法；（我觉得甚至不一定需要在 OB 上实现，或者 PG 上也有，能够用 PG 实现一些集中式的算法对比也许也行?

所以能不能设计一种什么新的、能够运用在分布式环境下的 视图/索引 选择算法，能够为这些本来适合于 OLAP 工作流的系统增添 一些 OLAP 的能力；能够算作 一个应用 场景？ 比如我们可以通过 什么 ob tools 之类的工具，提供一个运行物化算法的选项，设定好限制条件，就会自动运行算法选择视图并进行物化；

实验是否也可以 手动选择一些主要视图，实验对比看与自动选择的视图，查询性能差多少；加上不使用视图的实验对比，
（查询的生成问下有什么思路，或者使用什么数据集；

另外是否还可以对比一下，其他竞品数据库的 HTAP 能力，比如 TiDB 在相同条件下的性能？


--------------
实在不行的话，混合一些贪心或者机器学习的方法 试一试？ // hybrid solution 

在 search space 小于多少时采用  设想中 较慢 较优的机器学习方法；否则就快速贪心？


//
果然 是不是 能通过机器学习 的 方法，进行一些动态的视图选择？

// 
或者扩展一些 机器学习 的方法 到分布式场景中？


大规模分布式关系型数据库中的动态 视图/索引选择（算法）
// 实现一个 prototype？ 


最优化问题，早期的 mv 选择大部分都是贪心算法，后来有部分使用类似遗传算法/模拟退火算法 这类随机算法的研究，

// 现在的思路是：可以借鉴一些机器学习的文章，实现一个类似的模型？然后在实际开源系统中实现，与无 MV、某个贪心算法、（某个随机算法)的性能进行对比？

总之应用到分布式环境下，应该可以算作主要创新点； // 但是这样的约束会更多？

// 假设没有机器学习，工作量够吗？

强化学习选择视图：策略是要选择视图 Z 集合，动作是选择一个视图 Zj，reward 是计算得到的 utility 变更；

如果考虑在 OB 上进行实验的话，那么要考虑机器学习方法的实现难度 // cpp // 或者外置模型？ 但是还是很难 // 如果要做机器学习方法的话，就不考虑在 OB 上实现了；

//在选择候选视图方面，有没有能借鉴的地方？ // 比如只将 subquery 作为 candidate？ // 但是这样无论如何都会涉及机器学习；


// 不管是动态还是静态的； 动态的比较像工业的产品？ 或者能不能都实现一下做个实验对比？
我觉得能在 OB 上实现，就能把 分布式场景下 的应用 作为创新点；应用背景是 商业 OLTP 数据库，

// 是不是可以说：动态方法有更强的泛化能力，虽然简单；


如果要尝试一些基于机器学习进行视图选择的方法，我觉得有两个困难：
一是我对机器学习本来只有概念性的了解，比如一些论文里说用了这样结构的模型，能得到这样效果，我基本不知道为什么要设计成这样的结构；
二是OB是C++实现的，本来代码就复杂，再在上面做机器学习恐怕搞不定，时间也是个问题；


到时候在 OB 上做机器学习相关实验的话，就 Python 先把模型训练好，存到 OB 的系统表里；到时候直接硬编码从系统表读取，


那么，就是如果指定 workload，那么就使用 机器学习 先确定 candidate，然后分布式贪心的思路？ //或者直接贪心？
若不提供 workload，就考虑动态的实现？
// 一篇文章里包含两种方法和实现，应该内容差不多够了吧？

2012 survey
文献 5 是 P2P 的 动态选择 // 所以还没看
文献 29 是 Dynamat，大概是缓存了？ // 怎么讲不清楚？ // TODO：要能精炼的概括清楚；
文献 57 是 SQL Server 上的那个，只缓存视图中一部分经常访问的数据，并且缓存的数据可以根据设定的 LFU/Q2 等替换策略更换；


// 下载了两篇 view selection 的论文 2018 是物化查询执行过程中的中间结果的；2005 是...不知道
2018 selecting subexpression to materialize at datacenter scale
2005 selectiong of views to materialize in a data warehouse 

2000 The state of the art in distributed query processing.
// 之前的，还没看，似乎是关于对比 caching 和 materialized 的，caching 在启动时，pool 是空的；而 materialized 可以在启动之前（查询到来之前）提前选择需要物化的数据

1997 AutoAdmin 的论文，因为跟后面的 DTA 有关联，所以感觉得看看；而且 DTA（似乎 AotuAdmin 也是）可以一同选择 index/mv
// 连同 2018 的 DTA，看起来为什么像是一个 枚举 + 启发式规则 的算法？

// 想实现的还是一个 动态的视图选择
// 但是担心工作量的问题，因为动态视图选择放在这里可能只是 把原有的集中式应用到分布式数据库的场景下；原理上并没有太多创新点；
// 然后我想，本来对比实验里计划要做跟一些贪心算法的对比，那干脆也扩充一下，作为一个章节的内容来写；



// 研究现状
物化视图的应用最早出现在上世纪 八十年代？ // 时间需要核查 ， 来到九十年代后，一些主流数据库开始支持物化视图功能

// 索引选择算法，似乎不像视图选择算法那样考虑那么多的约束？比如空间限制，维护开销限制，还有分布式场景下的一些网络延迟限制、不同节点空间/延迟等不同的考虑


视图选择/索引选择算法，从暴力枚举开始，后来逐渐开始出现贪心和其他随机算法
而物化视图选择算法的研究在九十年代中后期开始得到较多关注，这一时期出现了许多基于贪心思想的物化视图选择算法； 

// 我的兴趣只在于看做出来之后，看看是什么效果，并不关注比之前的方法好多少；
// 但是论文里似乎必须要求有改进；


 GFS 带来的 硬件不可靠，需要通过冗余和恢复机制保证高可用；
  和 Spanner 开创的分布式关系型数据库先河？ 在此之前的分布式数据库，往往是采用分库分表或者中间件+单机数据库的方式，来实现分布式存储数据的效果，这样虽然能低成本地复用现有单机数据库配套设施、存量代码，充分发挥运维和数据库管理员的丰富经验，但是基本架构却还是集中式的，难以扩展。随着用户量、数据量的持续增大，运维将不得不手动管理越来越多的分区分表，；

  ，并且数据库本身的提供的事务 /--------/ 全局一致？ /--------/ 等特性也会因手动拆分的操作遭到一定破坏；

https://brewminate.com/a-brief-history-of-database-processing-since-the-1960s/
数据库历史：
1960s：数据库是 organization-wide，应用在企业内部，为了解决文件系统不足以支持企业数据管理要求而产生的，互联网还没诞生，此时数据库还是独立的一个系统

1970s：E.F.Codd 于 1970 年提出关系代数，从而发展出了关系模型数据库
依据关系模型，数据被组织成行和表的形式，能够避免一些 按照其他方式组织数据时 可能出现的错误（是什么呢？

1980s：因为 OOP object oriented programming language 的出现，也出现了 OOP database，但是不好用且企业不愿从关系型数据库迁移过来，因此没有流行起来

1990s：因为 LANs 出现，数据库开始朝 Client-Server 方向发展

2000年后，Internet

NoSQL

2010s：distributed databases

——————————————————————
数据是现如今信息化时代中，最有价值？的。

而作为承载信息的容器，数据库系统从上世纪六十年代出现伊始发展至今，已经从当初面向单个企业或组织、用于代替文件系统存储和管理数据的独立而功能单一的系统，发展为今天包括关系型数据库【】、/？？层级数据库【】、网状数据库【】？？/、包括图数据库、？？？、等的NoSQL【】、分布式数据库【】等在内的数据库大家族。数据库也不再是面向单个 离散的 企业或组织，而是通过网络与成千上万个节点或终端用户相连接，支撑着数千万、甚至上亿用户的应用服务。

———————— 历史
其中，关系型数据库在应用市场上一骑绝尘，虽然在诞生之初遭到不少非议和反对意见，但最终经过历史的检验后以压倒性的领先占据了 ？？ 的数据库市场 // 202x年，需要数据支撑。

也许正是这些反对，导致了 NoSQL 的///
在 2000 后，NoSQL // 因为什么而兴起的？

不过正如马克思的观点“历史总是螺旋上升”说的那般，数据库的发展也历经了波浪式前进的过程，在经过 NoSQL 一段时期的蓬勃发展后，业界发现抛弃关系模型的数据库在实际应用中出现了 xxxx 等太多问题。

人们开始重新回到关系型数据库的怀抱，
谷歌 2013 年发表了 Spanner，这也是工业界第一个大规模应用的分布式关系型数据库，，随后催生出一大批分布式数据库，如开源产品 CockroachDB、TiDB，阿里的 PolarDB ？？，蚂蚁的 OceanBase，等。

———————— // 机器学习呢？

随着互联网赋予数据的价值不断提高，业界对于数据的利用方式越来越丰富， ？？？？ 通过数据仓库进行的各类分析， ？？？ 运用机器学习的各类分析手段 ？？？ 等层出不穷。慢慢的，以往通过 OLTP 数据库系统进行事务处理、产生的数据经过 ETL 处理后导入数据仓库等 OLAP 系统进行数据分析这样的两套系统运行的方式，已经不足以满足如今数据分析任务实时性和用户体量迅速提升后的对系统扩展性、鲁棒性的要求了。

// 实时性的更多说明，为什么数据分析要求实时性更高了？
// 商机？调度？

为了解决这一问题，业界中开始寻求数据库的 HTAP 能力，即 Hybrid Transaction & Analysis Processing，整合 OLTP 和 OLAP ，在一套系统中同时提供事务处理和数据分析功能，在保证事务处理能力的同时，能够几乎无延迟的对最新数据进行分析。

而这样的融合，也意味着传统的  TP/AP 手段/特性？还是什么？？？ 要针对新的应用场景作出调整。如 TP 系统的存储方式通常为行存，而分析任务更多采用列存的方式来获取更优的读取性能，因而要在两种存储方式之间权衡，作出取舍；？？？

—————————————————— // 分布式呢？可扩展性呢？

物化视图也是 TP/ AP 系统中常见的一种通过持久化数据、减少重复计算开销的优化措施，在数据仓库中，选择适合的视图物化更是对整个数仓的性能有至关重要的影响。

在新的场景下



本文提出一种基于强化学习的分布式场景下视图选择方法，利用？？？并考虑？？？的约束，自动完成视图选择工作，来填补这方面的空缺。

目前看来的难点在于：虽然每个节点是对等的，但存储的 mv 需不需要考虑传输开销？ // 是不是可以不用？考虑 mv 也是对等的。 // 这些在分布式数据仓库中是需要考虑的问题，但是如果现在的场景是 OB 虽然 share nothing 但是对等的，是否就不需要考虑？

那就是约束，每个节点因为物理机器的不同，也许可以设置不同的约束条件？ // 能不能算作一个小创新点？

那么创新点在哪里？就是应用场景上吗？

--------------

两个风险：
1）OB 开源版现在对物化视图的支持不完善，就算是当时实习时也没有把设计的全部做完，实验可能无法直接在 OB 内核上修改并支持物化视图选择；
考虑解决方案：实验只是为了验证和对比算法的性能，那么考虑单独用 python 训练模型，并且跑数据集得到一个推荐的视图集合，然后手动把这个集合在 OB 企业版上创建；对比不同算法集合的运行性能；其他需要对比的算法同理；

1.1）这样似乎就不好说实现了动态视图选择
解决方法：要做似乎也可以，在开源上做，但是这个难度有些。。。又没有文档；如果说这部分做得比较简单，会有评审challenge吗？

2）对于机器学习的不熟悉，目前关于 RL 的方法几乎全部来自论文中出现过的部分，如 ERDDQN，等为什么要用这个模型，假如出现问题如何解决，被评审 challenge 怎么办，没有信心；

考虑解决方案：

3）另外还不知道如何用 mv 重写 query； rewriting！
// 思路上来说，要实现 rewriting，那么在生成逻辑计划时，优化器要能够检测当前存在哪些mv，然后判断根据开销选择执行计划（如果使用mv的开销更低）；那么后面的就不用再操心了；
所以这里需要修改的地方是：i）维护现有mv列表；ii）检测能够使用的mv；

4）需要的服务器先去问一下海天学长

5）到底如何实现算法分布式的扩展？

6）有没有可能做一个

主要参考 2022 ICDE 的索引选择，里面提到能满足动态 workload 主要是通过能够估算索引的维护开销，并且使用深度回归模型来学习并且估算，但是对于这部分的细节没有过多介绍；如果啃得下我也许就能同样说成是动态的，如果解决不了，那就说做了一个面向静态工作负载的分布式数据库视图选择算法；

姑且认为三个（比方说是三副本）节点数据是对等的吧

> 如何利用好已经拥有的？

---------

BigSub 中提到的，之前的 view selection，面向的 workload 似乎都是 “tens of queries”；（JOB也就一百多条查询

---------

size，规模

视图选择算法的 benchmark 多为几十到上百条查询

bigsub 能扩展到支持数万条查询的 workload；且分布式；

---------

那么，如果算法不能向 BigSub 一样分布式执行，能不能面向的数据规模再大一些呢？
// 也就是保留题目里的“大规模”，去掉“分布式”，但其实也可以保留，能解释得通说是要实现在分布式数据库上的；但算法本身是集中式的

这些 BOO BOW，bag of word
2020那篇用 LSTM 处理query生成候选的，这些技术点应该都可以作为相关技术里的内容（如果用到的话
