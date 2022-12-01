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

// Q1：疑问：物化的视图谁来选择？ 为何不是直接选择 subset？来作为 static 维护的视图？

abstract

传统的 mv 物化并维护视图中的所有 row，但本文提出有些 row 实际上并不需要访问和维护；

提出一种有选择性物化、维护视图中一部分（比如最常访问的 row）数据，并且物化的视图也可以进行动态的调整（可以手动或者 manager 自动调整；

在 SQL Server 上的实验结果表明，这极大的减少了存储和维护开销，同时提升了查询性能；


1 Introduction
更改要物化的 subset 可以通过修改 control table 完成；

举了个例子：一个 three-table join，替换成一个 view，但假设 view 中 0.5% 的数据拥有超过 90% 的访问频率（skewed，且这部分数据 changed seasonally，那么显然维护其余的数据收益不大；


这里似乎定义了一个 hotspot，然后系统只物化 hotspot 部分的数据（// 如何统计？ // cache manager 统计

另外，这里我们还可以通过把未完全物化的视图，当作 dynamic mv，通过 control table 在它未完全物化时，访问其中已经物化完成的部分数据；

main contribution：
1）动态调整和利用视图的 mechanism，更灵活，降低视图的维护和存储开销
2）扩展了视图的匹配和维护算法，来应用在 dynamic 场景下
3）引入 feedback loop 来动态调整 dynamic mv 的内容
4）介绍了 dynamic mv 的潜在应用场景；

sec 2 general form of dynamic mv
sec 3 control schema
sec 4 adapt contents of dynamic view
sec 5 expermiental results
sec 6 & 7 related work and conclusion

2 Dynamic Materialized Views

2.1 View Definitions
先定义基础视图 Vb，包括 SPJG（select，project，join，group-by）操作
dynamic view Vd 定义在 Vb 的基础上，通过 control table Tc 和 control predicate Pc(Vb,Tc)控制，

这里的 Tc 可以是普通的表，也可以是另一个 mv；因为是通过 control table 来定义（维护 dynamic mv 的，所以我们可以说是通过维护 control table 来维护/控制 dynamic mv 中的内容；

// control table 是否就相当于 bit map 的作用？存储 rows 的标识位？

view matching 和 maintenance 也许需要通过 control predicate；

2.2 View Matching

// 文献\[5] 中介绍了 视图匹配（view matching） 算法，视图可以用于整个查询，或者其中的子查询； // 是否要去看看？

要先将查询和视图的表达式转化为 normal form（范式？，来判断是否能使用视图计算得到查询结果

guard condition：在执行时测试 query 的结果是否能用某个视图来得到。

// 文献\[24] partially materialized views，证明啥？
大概意思是，如果要得到一个能够等效回答查询的谓词，需要原查询的 select-join predicate 能够推出 ==> control predicate 并上 基础视图的 select-join predicate
证明就看上面的文献咯

首先，guard condition 的检查，对于常规的静态视图而言，在优化阶段可以完成检查（因为静态视图包含着该视图定义的完整内容，不会缺失，最多因为维护不及时而内容陈旧（但这似乎也可以说是缺失？ 
// TODO：如何解释这一点？

动态视图前面提到，这样的检查需要部分推迟到运行时完成（为什么？ 
// TODO：是要读取 control table 的内容吗？这部分信息不能在优化阶段进行吗？

文章把 test 拆分为三部分，并将最后一部分放在执行阶段；

first part: Pq ==> Pc; 检查基础视图是否能够满足 query 的要求（也就是先假设我们这个 view 是完整物化的；背后的思想是：如果一个完整的视图不能满足 query，那么对应的 dynamic mv 肯定也不能满足； 

second part: add guard condition Pg； 证明 Pg ^ Pq ==> Pv ^ Pc ；// 后面文字没看懂，啥意思？ ==> 等效于，如果条件 Pg 能满足，那么 query 是否包含在视图内 ？；

thid part: verifying at execution time；
检查 guard predicate 是否存在于 control table 中；

// TODO：这两步就是判断一个查询能够用 动态视图 回答的理论推断，用于查询时做视图匹配的，可是咱怎么看不懂啊？

Theorem 1： query Q is covered by dynamic view Vd if ... there exists predicate Pg 满足以下条件：（如果存在；大概就是 Vd 中存在某一列，满足能够把原查询的predicate筛选的内容都筛出来，且这一列也要出现在 control table 中；即可

==> 符号大概是离散数学里的条件，if p, then q，前者为真，则后者为真；
那么这里的问题是：Pg是如何构造的？ 

Theorem 2： 如果 Q 的 predicate 可以转化为 disjunctive，那么对于每个离散的 predicate，都独立满足 theorem 1 的条件，则 Vd 对于离散条件的 Q 也算是包含； 

2.3 View Maintenance
dynamic view 的维护，因为只需要更新比较小的一部分数据，因而静态视图的全部维护更简便；但要做到这一点，需要对以往的视图维护算法进行一些改进（这些算法并不支持 exist 子查询视图的维护）

这一节介绍如何增量维护 dynamic mv；

如果 dynamic mv 定义里 control table 的 exist 对于 control column 中每一种可能的 value 最多只返回一个值 // 什么意思？表示什么？如果返回多个呢？
就可以当作常规的 join mv 来维护（Vb join Tc， base mv join control table）；
比如，将 primary key 作为 control column，那么就满足上述要求

如果 exists 可能返回多个值，则需要另外考虑 Vb 是否包含聚合：
// 为什么要区分单个值和多个值？要维护的是什么地方？
// 猜想：可能是因为维护的时候，我们不单要考虑 DV 中 row 的更新，也要同步更新 control table；所以如果 exist 返回的是单个值那么我们可以找到唯一对应的 control table row 来更新；但如果是多个值，就要保证这些 rows 都能更新才行；

case 1：对于 SPJ view，如果 output column 包含 unique key，那么可以将对应的 Vd 转化为 aggregation view 来进行增量维护；
// 按照 Vb 的所有列进行 group by // why？
// 以及 什么是 unique keys？ 主键？ // 可以唯一表示某一行的 候选键，可以包含多列，一行只有一个候选键作为主键，但可以有多个候选键
// TODO：那么这样的 view 为什么在 exist 会得到多个值？

create view Vd as
select Vb .*, count(*) as cnt
from Vb , Tc where Pc (Vb , Tc ) group by Vb .*

通过 group by 消除 duplicated rows，并添加 count；


如果不包含 unique key，则需要进行额外的 join 操作 来避免引入 dupliates；
对 Vd 进行 self join；

case 2：如果 Vb 是 aggregation view；如果 Vspj 部分包含 unique key，那么可以将 Vd 重写为 
create view Vd as
select Vb .*
from (select Vbspj .* from Vbspj , Tc
      where Pc (Vb , Tc ) group by Vbspj .*)
group by G

通过 from 后这部分的子查询来去除 duplicate；
类似的，如果 Vspj 不包含 unique key，那么可以用 case 1 中的方法进行 self join；

// TODO：有一点基础问题没搞懂的是：这里的视图维护到底是什么步骤？为什么都在强调要去除 duplicate，这些 duplicate 是从哪里来的？对增量视图维护有什么影响？

// 这一节的思路似乎是：现有对于 SPJG 视图的增量维护算法已经十分完善，但是却没有能够支持 exist 子查询视图 增量更新的算法，而 dynamic view 是包含 exist 的；因此我们想办法将 exist 子句转化为 SPJG 视图来进行增量更新，就能够复用以往的增量更新算法；因此这一节就是介绍如何转化 exist 子句的；

3 Control Schema

control predicate 有多种类型；

Equality Control Tables: 在一个或多个列上的 equijoin；这类 control tabe 只能用于 join 的视图 

Range Control Tables: 使用 range control 的 dynamic mv 可以支持 range queries 或者 点查询；

// 看看这里的 guard predicate 例子，就是 table 中需要包含元素的补集；
range control table 指定某个 range；
这里提到也可以只指定上界或者下界，这样的 control table 只包含该 上界or下界 的一条记录，并且能够用于 single bound、range、equality 的查询；

Control Predicates on Expressions:
control predicate 也可以应用在 base view 的 function 或者 expression 上；

// 例子就是在 exists 部分的 where 条件中，可以使用 function 或者其他 predicate 来过滤；

More Elaborate Control Designs:
一个 dmv 可以使用多个 control table，它们的 predicates 也可以组合使用；// how？// 通过 exist 部分使用 and 谓词连接两个 exist 条件；

另外，不同的 dmv 也可以共用 control table；也可以将 一个 dmv 定义在另一个 dmv 之上；

比如，将 customer 表中 hot segment 部分定义为 DV1, 同时我们还想查看这部分 customer 的 order，那么可以在 DV1 之上定义 DV2 来存储 hot segment customer 的 order 部分；

4 Control Table Management

control table 可以当作普通表 base table 来更新；

但需要根据 DBA 定义的 policy 来决定要丢弃哪些 rows；文中设计的更新方式有 automatic 和 manually 两种；分别介绍：

Automatic Caching：就使用某种缓存替换策略；

实现上这个 cache controller 是 in-memory 的，常用替换策略包括 LRU，LRU-K，Clock，GClock，2Q（这是什么？

但分析发现，这些替换策略并不适用，没有将 替换 的开销纳入考虑，因此文中基于 2Q 算法设计了自己的替换策略； 
// 2Q 算法 参考文献 \[14]；
- 使用两个队列，一个用于 admissin，另一个用于 eviction；
- 在 new row 第一次出现时不考虑插入 control table
- 检测两个队列中存储的 每一行 的最后访问时间，
- 如果队列已满，如果要将新 row 插入，那么它的 热度必须大于 队列中 coolest 的元素
- bitmap + hash 减少内存占用；

实验表明，这样的设计 space efficient 和 更高的 hit rate
这样的自动缓存在 数据库中间层 中也适用

Incremental View Materialization：如引言中提到的，也可以使用 dmv 的机制来增量更新 那些维护代价比较高（时间长）的 mv；

通过 range control table 完成，可以逐渐扩大 range 的范围；在 view 完全更新完成之前，可以把他当作 dmv 来使用；// 感觉是个无关紧要的应用；

dmv 可以用于聚合 hot items，blah blah 其他应用；
// 细节见 文献 \[24] // 是否考虑看一眼？

5 Experimental Results
实验里只实现了 equality control predicate ////

为了 skewd access pattern 设计的 dynamic mv； // 这种场景比较适合

- 因为 dmv 不能用来响应每一个 query，所以实验目标是证明 dmv 对于 skewd query 的表现不比完整 mv 差；
- 在内存空间受限的情况下 dmv 的性能表现； // 因为 cache controller 是 in-memory 的
- 最后检查 maintenance overhead

5.1 Query Performance
实验设计了一个 three table join，三个基础表大小1.5G，对应的完整 mv 大小约 1G；给 DMV 预留的内存空间是 50MB 左右；

对于 90% hit 的 DMV，表现性能表现与 FMV 相当；如果 hit rate 进一步提高，那么 DMV 的表现会上升；

DMV 和 FMV 的查询性能当然都远优于不使用 MV；

5.2 Processing Fewer Rows
上面的实验是按照 control column p_partkey 来聚簇的；/// 不太懂

如果不是这样的？ // 改变实验条件
理论分析 DMV 表现将会比 FMV 更好；


// 这个实验只跑了100次是不是太少了点？
并且令 DV 的 control table 总是包含需要查询的 rows； 变量只是其他不需要的 数据行 的数量（与 FMV 总是加载所有数据行进行对比

在 buffer pool cold start 的情况下， DMV 包含的数据越准确、无用数据越少，就能节省越多开销（这是当然的，但更需要解决的问题应该是：如何令 DMV 包含的数据块更精确？ 这显然很难，在这种架构下

// 这样的实验感觉意义不是很大

5.3 Update Performance
使用 FMV 5% 大小的 DMV，每次更新只更新 TPC-H 某个表的一列数据，总共的修改包括3张表；

两种场景：
1 是 uniform update，random k/v
2 是 skewed updates，skewed k/v

实验结果来看，因为 DMV 包含的数据非常小（ FMV 的 5% ），所以显而易见 DMV 的维护开销也远远小于 FMV；

对于 uniform update，DMV 开销在 FMV 的一半以下；skewed update 的节省开销 DMV 没那么多了（为什么？


6 Related Work

// 可以再详细看一下 这部分

view matching

incremental view maintenance

partial indexes

dynamic plan


// TODO：SQL Server 的 @名字 是干吗的？