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

物化视图的维护，通常采用 eager maintenance 的方式，即在更新基础表的同一个事务中更新视图；
文章认为这些开销不应该放在update中，因此提出 lazy maintenance，在系统空闲或者查询需要使用视图时，才对其进行更新。

好处是有机会能将多次 update 合成一次更新，update 的响应时间变短了，也明显降低了 update 的维护开销；
在 SQL server 中实现（2007,微软发的论文

1 Introduction

为什么要使用 lazy maintenance？
如果有许多 small update，更新时维护视图效率比较低，而且更新的响应事件也比较长。

一些数据库使用 deferred maintenance 策略，需要用户请求更新视图时才对视图进行维护（Oracle？
缺点：
1）这样的视图无法保证最新的，优化器可能使用旧的信息构造执行计划； 
2）视图维护工作对用户来说不再是透明的，也不是自动的
3）用户需要知道 query 需要使用那些查询，和判断这些视图是否 up to date（并负责更新视图

因此文中需要一种能够减轻 update 开销和自动维护视图的方式，因此实现了 lazy view maintenance，

update 时先存储被更新视图的更新信息（而并不直接更新视图），在系统空闲时，自动发起更新视图的任务；如果在视图完成自动更新之前，有查询需要使用未更新的视图，那么系统将立刻执行更新，执行完毕后才返回查询结果，但这次更新只涉及查询需要的视图，其他的未维护视图并不需要在本次更新中考虑。（可以继续等到系统空间，或者被其他查询使用时）

update faster，更快释放锁；实现上采用了 row version（多版本？），简化了维护操作。
文中还引入了 Condense operator和一系列优化规则，来合并对相同行的多次更新操作


贡献：
1 lazy view maintenance
2 row versioning
3 通过合并 maintenance task 减少维护视图的开销
4 通过后台进行检测未更新的视图
5 在 SQL server 2005 中实现了原型并进行测试，说明可行性

sec 2 lazy maintenance 的概要
sec 3 维护的基础算法？合并多个更新事务
sec 4 Condense Operator 和产生更高效维护计划的优化规则
sec 5 实验
sec 6 问题讨论（也不知道啥问题
sec 7 related work
sec 8 conclusion

2 Solution Oerview
简要介绍 SQL Server 2005 的 row versioning 和 snapshot isolation

2.1 Background Information
SQL Server 2005 使用 version store 来存储 record 的旧版本，但不用于持久化，仅作为 online index build 和 SI 的用途。

每个事务开始时持有一个标识符 transaction sequence number (TXSN)，和提交时申请一个提交序列号 commit sequence number（CSN），每个 statement 持有 STMTSN。

每个 record 包含 TXSN 和 STMTSN，同时 version store 为每个表、视图、索引维护一个 version chain，能通过 chain 来找到旧版本的数据。

给定 TXSN 和 STMTSN，能在 version store 中找到事务或语句 “开始或者结束” 的数据版本。

version store 同时也记录所有的活跃事务和他们需要的版本，当没有事务再使用某个旧版本之后，这部分数据可以被回收。

SI 保证所有读操作看到数据库是一致的状态，读不阻塞写。

2.2 System Overview

架构图简要来说，基础表更新之后，不再直接更新视图，而是把需要更新的数据存放在 Delta Table 中，加入任务对列，然后由一个低优先级的 Maintenance Manager 调度更新。
（在Sec 6 中会对比其他实现方式）

Delta Table
对基础表的更新操作被记为 Delta rows，然后我们将这样的 delta stream 转化为 action column ACTION 记录是增删改的哪一种）以及编码记录在基础表上改动了哪些数据（这一步倒是没有明确提及

（好像也就跟之前在OB做的差不多）
一次更新会被记为一次 delete 和 一次 insert 操作，在文献\[7]中有更详细的介绍。

delta table 中还记录额外的两列数据 TXSN 和 STMTSN，标识哪个事务哪个语句产生的这一行 delta 数据。这两列数据将用于构建 maintenance expression。

delta table 根据 TXSN，STMTSN，Action 和 基础表的主键来聚类（标识数据行？

Maintenance Task
一个更新事务提交之后，每有一个与该基础表关联的视图，就会产生一个对应的 task。

maintenance task 指定需要更新哪个视图、对应的基础表、对应更新事务的 TXSN 和 CSN、STMTSN 和任务当前的状态（pending/inprogress/compeleted)，TXSN 和 STMTSN 用于确定具体的 delta rows（存在Delta table 中），CSN 用于决定 task 的执行顺序。

Maintenance Manager
负责跟踪活跃的维护任务，和它们需要哪些版本的数据，并负责生成 maintenamce job 来调度执行。

维护了一个哈希表来存储 maintenance task，拉链形式，每个链上按照 CSN 升序存储。

为了能够判断需要哪些版本的（这里指的是数据吗？）和 delta streams，manager 为每个更新事务维护了一个处于 pending 状态的 maintenance task 列表，当列表中任务全部完成后，就能判断该事务涉及的 delta stream 不需要了。（垃圾回收旧版本数据也是该组件负责吗，是否每个 record 独立决定要保留哪些版本的数据？

Task Table
维护任务同样需要一个全局的 view-maintenance task table 来存储，主要用于故障恢复。维护任务会作为（更新）事务的一部分存放到表里，然后在执行维护任务时从表中删除。（在分布式环境里这种操作并不方便，但是不是也没什么办法如果这么设计的话）

2.2.1 Update Transactions
// 基本在介绍架构图里的流程，

// 但是不太明白的一点是：如何保证系统故障时，maintenance task 没有丢失也没有多余呢？是否把 Task Table 的写入也包含在事务的执行流程之内？ // 也许是的，毕竟集中式多表的更新开销并不会多太多。

2.2.2 Lazy Maitenance
周期性唤醒 manager 查看是否有 pending 状态的任务，有的话则构建一个后台 job 开始调度执行（ job 总是按照原始事务的执行顺序开始执行）

manager 可能会将多个 task 合并为 一个 job 执行。 Job 执行完毕后，manager 将对应的 task 从列表中删除（同时也删除持久化的 task table 中对应的 entry）并释放对应的 delta rows

删除 view 则对应的 task 也将被清除。

2.2.3 Query Execution

在查询计划执行之前，需要检查是否该计划使用的所有 view 都已经 up to date（不存在 pending maintenance task） 称为 on-demand maintenance

（查询 abort 也并不会导致原本的维护任务 roll back（毕竟本来就是要接着更新的，只要执行过程没出岔子就行

一个略微复杂的场景是：在一个事务内，我们先更新某些数据，随后立刻执行对这些数据的查询操作。这些查询应当需要看到先前事务内做的修改（因此我们需要更新视图），但是这些视图的更新我们不确定能够持久化，因为当前事务有可能abort（

我们通过以下方式处理这一问题：假设执行的事务需要引用视图V，在该事务执行之前，先通过 maintenance manager 发起一次对 V 的更新（作为另一个事务），然后如果本事务再对 V 进行了更新，那么将直接将 update 应用到 V 中。这样能确保事务的更改能反映在 V 上，同时也通过 current txn 的上下文保证如果 current txn 回滚，V 中的修改也能回滚。

//（那么就需要识别哪些执行计划会先更改视图，然后再查询视图；毕竟不能对每个事务都在开始执行前强制维护一次所有引用到的视图吧）


2.3 Effect on Reponse Time

Lazy Maintenance 缩短了 update 的响应时间，但是 query 的响应时间有可能变长。


3 Maintenance Algorithm


3.1 Full and Partial Maintenance Tasks
// 重复了一遍 lazy maintenance 的过程

full maintenance task：事务T对应 TXSN 我们记为 T.TXSN，假设 T 的第三条语句 插入行 delta R，修改表 R，存在视图 V 引用表 R 和 S。那么此时，在 T 没有修改 S 的情况下，我们只需要记录 TXSN = T.TXSN 和 STMTSN = 3；如果先前的语句没有修改 S，那么 STMTSN 此处也可以不记录，因为第三条语句看到的 S 的版本跟事务开始时的版本一致。

partial maintenance task：如果接着上述例子，第四条语句引用了视图 V，根据 sec 2 的介绍此时将会把第三条语句的修改持久化（一旦current txn 提交那么修改就是持久的），随后出现第五条语句更新了 S，此时 delta stream 记录的 TXSN = T.TXSN，STMTSN = 5；此时 STMTSN 能表明第5条语句之前的更新已经被应用到表中，我们只需要继续应用第5条语句之后的修改即可。

3.2 Normalized Delta Streams
介绍一个 observation

假设事务 T 修改了两张表 R 和 S，产生的修改设为 ∆R1,∆S1,∆R2,∆S2...，按照该顺序应用 delta stream 之后，R和S表将从初始状态 R0和S0，变为 final state R1和S1。那么重新排列这些 delta stream 为 ∆R1，∆R2，...，∆S1，∆S2...，按照这样的顺序应用修改，同样能够得到 final state R1和S1.

因此我们可以将 ∆R1，∆R2，... 合并为一个大的更新 ∆R，得到 delta stream ∆R. 对 S 的更新也可采取同样操作。此时我们可以分别得到对两个表的 delta stream，而这两个 delta stream 的应用顺序先后对 final state 并不会有影响。

Proposition 3.1： // 总结了以上的观点，此处略 // cp原文
Two sequences of delta streams affecting
a set of base tables are equivalent if both produce the same
final state when applied to the same initial state of the base
tables. Any sequence of delta streams can be normalized
to an equivalent sequence of delta streams consisting of one
delta stream for each affected table.

总结：存在许多等价的 delta stream，我们可以根据需要任意选择来应用更新。

3.3 Computing View Delta Streams
本节介绍如何计算 Delta Stream ∆V

维护 aggregate view 可以在 delta rows 部分计算需要聚合的数据，在文献 \[7] 中有更多展开。
这里主要介绍 Join View？

n-way join view V = R1  R2 ... Rm 这公式就没看懂？是这视图本身是 M 个基础表的 join ？
// 大概是的

所以 ∆V 就是（假设对其中一个表 Ri 进行了修改，那么就是 ∆Ri 跟剩下的其他表的存量数据进行 连接？
// 参考文献 \[4] 但是页数有些多

所以 ∆V 就是每个表 Ri 的增量数据 ∆Ri 跟其他表现有的数据挨个应用（文字描述不够清楚，看看原文公式
\[10] 中有类似的公式；

并且我们给 delta view 中每一步加上了 Step Sequence Number（SSN），*实际上是每一行变更的时候会记录它对应哪一步
（等式对partial maintenance task也成立（要调整

------------------
。。。有些没读懂的地方

3.4 Combining Matintenace Tasks

延迟维护视图，就可能导致维护任务的堆积。manager 需要保证这些任务按照原事务中的提交顺序来执行。

一个 naive 的思路是按原本的任务序列逐个执行 任务，但如果更新中包含许多 small update，那么这种更新方式的效率就不是那么高。

对于这种情况，很容易想到我们可以将某些更新任务合并成一个大的更新（或者说批处理的思想，好处是：
1）合并后避免多次调用计算相同的 maintenance expression
2）可以省略一些对同一行数据的重复更新（直接采用最后一次更新的数值（详情见 sec 4


假设一个视图 V 有一个长度为 l 的维护任务队列，那么我们可以将这些维护任务合并为 以其中事务最早的开始时间戳 TXSN 为开始、以最晚事务的提交时间辍 CSN 为结束的大事务。 // 这里是简要概述，

合并成大事务之后，我们也只需要获取这个大事务开始前的数据版本和 after-version（这个需要吗？）加上 这期间的 delta changes

这里 task 的执行顺序是按 SSN、TXSN、STMTSN 聚合之后的顺序来排列的，这可能与事务的提交顺序 CSN 不同，但是并不影响最终的执行结果。理由是SI能够保证两个提交事务（修改了同一个数据的），他们的提交顺序必须依照事务开始的顺序；如不遵守，就会因为发生冲突而导致其中一个回滚。（这一点由丢失更新的检查确保
因此按照SSN、TXSN、STMTSN的组合来进行排序，就能保证视图更新结果的正确性（仅仅TXSN不够吗？

// 有一段没看懂，：维护任务也不总是能够合并的，一些中间版本可能会丢失（什么情况下？//在说什么勾八东西
创建一行数据的新版本时，旧版本数据只有在被其他活跃事务引用的情况下才会保留，如果没有pending或者活跃事务引用，那么（可以safely combinme？ // 这到底说的是什么勾八东西

3.5 Scheduling Maintenance Tasks
维护任务调度分为 后台调度 和 按需调度。

Background Scheduling
后台调度，虽然可以选择任意视图进行维护，但是还是需要在几个目标中进行取舍：
1）减少 query 等待视图维护的时间
2）维护工作要高效
3）最小化资源使用

为了满足1），可以根据“视图期望被更新的时间长短”，作为维护视图的优先级；
对于2）,如果一个视图有多个 pending task,manager需要决定将多少个 task 合并为一个 job 执行:越大的job 执行效率相对越高,但是时间也会越长。
对于3），因为 delta stream 和旧版本数据按照时间顺序进行回收，因此一个旧的 task 可能会阻止后续空间回收，因此越老的 task 应该获得更高的优先级。

其他的考虑也许有：将相同视图的维护任务排在一起，以便能够利用共同的 subexpression  和 buffer pool缓存。

文献\[6,15] 中的做法，可以参照，将维护任务分为两个阶段：计算 delta 视图 和 应用 delta 视图；这能够进一步减少资源竞争。


On-demand Scheduling
被 query 触发的 task 调度会立刻执行，并与 query 具有相同的优先级（因为 query 需要等待 task 执行完毕才返回结果）

如果查询从视图中访问的那部分数据并没有被更新的话，那么可以不立刻触发调度。因此可以先检查 query 需要的数据是否有被后续的事务修改。

比如：可以将 查询的谓词投影（project） 在 基础表上，并用同样的投影来扫描对应的 delta table，如果扫描并没有在 delta table 中返回数据，那么可以推断 本次 query 需要的数据并没有 pending update

4 Condensing Delta Streams
视图的增量更新∆V包含4列额外数据 Action，SSN，TXSN,STMTSN。本节主要介绍action column 如何应用，以及其他的优化策略。

SQL Server 中通过 StreamUpdate 运算符来接受 delta stream 并将这些修改应用到视图中。

至于 Action 列的使用，我们先将 delta rows 按照 clustering key 进行排序（大概就是上述的几列额外数据）保证对同一行的 delta row 都是相邻的，而 delete action 比 insert 对应的值更小，会出现在 insert 前。

接着通过 Collapse op 来合并 Delta Stream：如果两行 delta row 的 key 相同，且第一行是 delete 第二行是 insert，那么可以将这两列替换为一个 update（啊？这不是本来就把 update 拆分为 delete/insert吗？

简化完毕的 delta stream 由 StreamUpdate 应用到视图。

\[7] 中有关于上述操作能充分利用数据局部性的更多介绍。

4.1 The Condense Operator
即时更新的视图维护策略里，∆V对同一个最多包含 row 的两条修改（update 导致的 一个 delete 和一个 insert

但 lazy maintenance 中就可能出现多个 delta rows，如果按顺序应用，虽然能保证最终结果的正确性，但是不够高效；这里引入了 Condense Operator。

首先stream需要按照 clustering key 排序，保证对相同视图的 delta row 都凑在一起。
Condense OP 根据 一个 group （就是对同一个 row 的所有修改）中第一条和最后一条 delta row 的类型，来生成化简后的  output stream，规则如下：
first Insert：
  last Insert：last delta row // 要知道最后插入的是什么
  last Delete：Output nothing

first Delete：
  last Insert：update，output update delta row for full condense，first delete and last insert delta for partial condense
  last Delete：last delta row（？why？）// 我们需要知道这一行最后是被删除了

4.2 Partial Condense
上文表中 partial condense的内容，（对比一下 sec 3 里面的 


同样，跟 那篇 Oracle 的论文类似，不需要先对 ∆R join S 连接，再压缩相同的行；而是可以先对 ∆R 进行压缩，以便能够跳过那些对相同行的重复修改；

// 3（c）没看懂？为什么需要是 nested loop join？

对于两张表都产生了修改的情况，同样可以先做 partial condense；

5 Expreimental Results
在模拟环境下，比较了 update 和 query 的响应时间，以及 lazy maintenance 和 eager maintenance 的维护视图时间开销；最后对比了 最坏情况下 的表现。

5.1 Update Response Time
定义了一个 join 和 aggregated 的视图 // 此处略

结果当然是 lazy maintenance 在更新1,10,100条记录时表现更好（update response没有明显增加

维护多个视图时，类似结果

都是可预见的。

5.2 Maintenance Cost
总的维护开销比起 eager 策略要大，但响应时间短（这么说来系统的总体负载似乎会上升一些？）

5.3 Query Response Time

如果查询与更新之间间隔足够长，那么 lazy maintenance 策略下二者响应时间都很短；但如果间隔不够长，那么查询的总开销当然会大于 eager maintenace；


5.4 Multiple Updates

对（更接近实际场景的）多条更新，使用了 condense 的 lazy maintenance 整体的维护开销更少（因为压缩之后，对于修改数据比较分散的情况，主要减少的开销是不用重复创建 维护计划；对于修改数据比较集中的情况，主要减少了对重复数据行的更改操作

5.5 Oerhead for lazy Maintenance
与5.2类似，maintenance task 的持久化，多版本数据的维护和清除，存储 delta stream等都是额外的开销，系统在 update 和 lazy maintenance 的总时间当然也长于 eager maintenance，不过 lazy 的 update 响应时间远远小于 eager。

6 Future Consideration

6.1 When to use Lazy Maintenance
以下因素出现时，可以考虑 lazy maintenance：
1）update 的比例高（需要 update 的响应时间更短），以及 update 之后立刻出现 query 的概率小
2）update 的规模相对于维护视图的开销来说比较小（每次更新一小部分基础表数据，却几乎要更新整个视图，比如一些聚合视图
否则还可以继续使用 eager maintenance（基础表更新少，更新后常立刻跟随查询操作等）

6.2 Recovery and Error Handling 
维护任务执行过程中出现故障，则已经执行的部分会被 回滚，并且通过 task table 中的信息重建 task manager；

存在 lazy maintenance task 不能完成的情况？ why？
mv 必须有一个 unique clustering key，如果这一约束不能保证则任务可能失败；
// 所有deferred asynchronous view maintenance 算法都有的问题？\[6,15]？？why

提供两种解决方法：

6.3 Alternative Approaches
文中采用的是通过 ersion store 存储基础表修改的数据记录，来完成后续的视图维护工作；其他方法可能有：
1）把变动的数据存在delta table（或称为 table log）
2）从恢复日志中提取数据（expensive
3）从 version store 中恢复 delta 数据（expensive

// Q：version store 和 delta table 的区别/用途到底是？
// 是不是说 version store 其实是指基础表的多版本数据？



7 Related Work
MV 在 Orecle、DB2、SQL Server中都有实现；

文献\[4,10,9,13,12,5,11] 都提出了视图维护算法，但都是 eager 维护策略，使用 delta paradigm 来存储更新的 tuple；

\[6,15] 提出了 deferred / asynchronous 视图维护方法。\[6] 的目的是最小化 view downtime，

\[17] 另外一些视图维护的优化工作；


8 Conclusion


