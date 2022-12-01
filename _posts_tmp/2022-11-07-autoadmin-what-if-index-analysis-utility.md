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
建立索引能够大幅提升数据库的查询性能，因而索引的选择至关重要；这项工作通常由 DBA 负责。

这里提供了 SQL Server 上实现的一项索引量化分析工具，能够帮助 DBA 分析一个“虚拟”索引（hypothetical indexes）能够带来的增益（没有实际建立物理索引结构），也称为 what-if indexes （后续可能包括了mv之类的评估，毕竟类似的结构

impact analysis tool

// Q：这种功能现在是不是大部分数据库都会支持？为什么现在还有很多数据库建立索引，是依靠 DBA 的经验？

1 Introduction

比如DBA 希望能知道：
1）过去三天的哪些 query 和 update 会因本次 index 改动而变慢？
2）哪些 query 会便快？快在哪，快多少？


1.1 Overview of Architecture

configuration refer to a set of indexes, and the sizes of each table;

workload: a set of SQL stetements;

hypothetical configuration analysis(HCA) engine 支持
1）模拟 hypothetical configuration
2）分析使用该 configuration 的影响；

但是不公开源码，只提供接口； （当年，现在啥样暂时不清楚

能提供的分析包括：
1）每个查询的数量
2）configuration 需要消耗的空间
3）添加索引后 workload 中将被影响的 query


sec 2 related work
sec 3 configuration simulation interfaces
sec 4 summary analysis interface & implementation
sec 5 future work & conclusion


2 Related Work

虽然索引选择的研究从 70 年代就开始了，但是在98年之前并没有工作研究如何评估索引和数据库大小改变对查询产生的影响；

\[12] 提出了使用视图来模拟数据库，但是需要实际执行该查询，开销很大；

相比之下本文的工作并不需要实际执行，而只是使用 estimation of cost;

\[4,6]通过部署本文中的工具来进行索引选择； // 那就是关于索引选择的文章，暂时不看

\[2,5,7,9,11] 的大部分索引选择工具 都没有支持 what if calls
// index selection tools do not stay in step with the optimizer（意思是索引选择工具运行结果无法很好被优化器利用？

\[3] 中有更多讨论；

3 Simulating Hypothetical Configurations

foundational concepts

a) workload，一组 SQL 语句；数据库通常可以根据日志生成 workload
b）hypothetical configuration，一组索引（满足 schema 约束）；这里的 configuration 可能会包含 database scaling value，表示 configuration 的 size 与当前数据库的 size 之间比值（大概意思；
总之是用来判断 configuration 的大小的；
c）estimation of projected changes，HCA能够评估 config 对当前数据库的影响，一个是能评估 使用 what if config 之后查询的开销，另一个是能够评估 索引的使用情况（index usage

3.1 Interfaces for Hypothetical Configuration Simulation

提供的接口：
A）Define Workload
B）Define Configuration
C）Set Database Size of configuration
D）Estimate Configuration of workload
E）Remove Workload/Configuration/Cost-Usage

// 关于各个命令的详细介绍这里就略去，反正命名都很直观；

唯一值得一提的可能是，Estimate Configuration 命令，对 需要评估的 workload 中每一条查询都会返回一个结果，包括这条查询的开销是多少（相对于整个 workload 的相对值），以及使用了哪些 index；

但是并不知道如何实现的；

3.2 Implementing the Hypothetical Configuration Simulation Interface

介绍 Estimate Configuration 的实现；（好，这次确实是我想看的了

首先明确我们不能实际更改 物理的 configuration；


思路来自于 optimizer 评估的查询开销并不是“实际”的执行开销，而是基于 statistical info 中统计的 index 覆盖 column 数量，来决定是否使用某个索引；

因此只要能够收集 1）索引对应 column value 的直方图 histogram；2）密度
收集这些统计信息就能支持 优化器 来评估 hypothetical index 在执行计划中的表现；

因此 Estimate Configuration 的执行步骤为：
1）创建所有 hypothetical index
2）请求优化器：a）将索引的选择范围限制在给定的 configuration 内；2）根据给定的 scaling value 更改调整需要评估的 table 和 index size；
3）请求优化器生成最优的执行计划，并收集：a）查询的开销；b）使用到的索引；

// 也就是说相当于把这些任务都丢给优化器来执行了？那么优化器是怎么做到上面那些分析的啊？



3.2.1 Creation of Hypothetical Indexes
page-level sampling algorithm，用来收集统计信息；

初始从 m = 根号n 个 sample 开始，收集 m 个 别的 sample，如果 收敛 则结束；否则继续添加 sample， // 我怎么看不懂捏？

// 为什么叫做收敛？什么情况下才叫做收敛？

sampling 是什么？ // 抽样的方式得到数据密度，数据分布的直方图？


来看样例（通过 sampling 创建 hypothetical indexes 同时不影响优化器的估计，是我想看的）：

// 大概了解了，优化器评估执行开销需要表和索引的统计数据，这些数据可以全表扫描收集，也可以通过 sampling 的方式抽样得到；这里 抽取 5% 的数据样本，减少全表扫描的开销同时也能很好的概括数据的分布情况，可以当作真实的 statistical info 来使用。

// 给出的实验数据表明， 1%，3% 的 sampling 还是有些偏差的（1%的估计偏差最大可能达到20%，但是5% sampling 的偏差不过 4%，应该算是可以接受的范围


3.2.2 Defining a Hypothetical Configuration

通过 HC mode 传递 hypothetical index 需要的信息，包括：1）生成 query plan 时指定 configuration 的 index 集合；2）设置每个 table 的 base index；3）table & index 的 size

3.2.3 Obtaining Optimizer Estimates

只写到数据库API 提供 “no-execution” mode，能提供查询计划、优化器估计的开销；

猜想应该是通过执行计划，估算每个步骤需要的运算+IO次数，然后根据现有的数据可以推测；比如优化器知道执行对表T的全表扫描需要一万次 IO，开销为 X；那么预估现在的执行计划需要进行十万次 IO，因而可以估算这些代价；

3.3 Maintaining Analysis Data Tables

我们在3.1中讨论了如何生成（查询和索引）统计数据的 analysis data table，现在来讨论它们：1）如何命名？（这重要吗？；2）存储在那里？

3.3.1 Analysis Data in System Catalogs
?
不知道他这段有什么意义，凑字数？

3.3.2 Analysis Data in User Specified Tables
用户可以将得到的 Analysis Data 存储到制定的表格中，也是 HCA engine 提供的功能（那还有什么好说的？有必要写吗？

4 Summary Analysis
也就是提供一些统计信息啦，譬如：每种类别 query 数量都是多少、每个索引被使用的频率是多少、每个列出现在 selection 中的频率是多少等；这些就是文中说的 analysis data；

后面就是描述 HCA 如何提供接口来让 DBA 查询到这些信息的；（似乎没有看的价值？

4.1 Conceptual Model for Summary Analysis

主要包含三类：
1）对 query 的 Workload analysis
2）对 index 的 Configuration analysis
3）对 query 和 configuration 之间关联的 Cost and Index Usage Analysis；

4.1.1 Objects and Properties
每种分析数据的对象当然就是他们名字中包含的那样，每种对象也许包含不同的属性；（让我看一节内容 看看到底都是什么）


4.1.1.1 Properties of Queries

atomic value: 
query type (Insert/Delete/Update/Select)；
Group By clause
Order By clause
whether has nested subqueries

properties with list values:
referenced table
required columns
selection condition on columns
join condition
equi-join condition

看这些有啥用？

4.2 Summary Analysis Interfaces

4.3 Examples of Summary Analysis and User Interfaces

4.4 Implementation of Summary Analysis Interfaces

基本来说就是将用户需要的 analysis 转化为 SQL 在 analysis data table 上执行；



4.5 Example of a Session

4.5.1
4.5.2

4.5.3 Exploring “what-if” scenarios
总之就是可以对比某个 hypothetical configuration 和 current configuration 对 workload 产生的影响；


5 Conclusion

这个工具（index analysis utility）能够帮助 DBA 分析索引的选择（那么一些索引选择工具和算法到底如何取舍？// 作为参考值？


// TODO：也许应该重新归纳一下这篇论文，提炼一下它到底如何实现这样的评估的


