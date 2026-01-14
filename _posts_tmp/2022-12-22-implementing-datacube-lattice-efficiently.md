---
layout: post
title: "[]"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - database
  - paper
---

abstract

 datacube lattice 

 用来表达视图之间的依赖关系

 提出贪心算法来选择视图

 // 带着问题：1：贪心的思想是什么
 // 2：datacube 之间的关联是什麽

 1 Intro,

 3 Cost Model

 4.1 Greedy algorithm


 ------------

 1.1 the Data cube 

multidimension datacude 2-D, 3-D; // example?

好像是 基础表 按照 attribute 聚合的 维度？

这些什么 part / supplier/ customer 都是基础表上的 列？

维度大概类似表的个数？ 3-D 是 三个表的 join

1.2 Motivating Example

不考虑索引，

考虑空间限制，


Multi Dimensional Data Processing( aka. OLAP)

假设这些 datacube 存储在 系统的 summary table 中
// 什么是 summary？


select Part, Customer, SUM(SP) as SALES
from R 
group by Part, Customer

这个 SQL 里 supplier 是被全部聚合的，所以这个我们可以记录为 View_S

// 这里的例子都很简单，只有几个 group？ 几个简单的物化视图（虽然数据量可能稍微多一些



2 Lattice Framework

3 The Cost Model

3.1 The Linear Cost Model

因为只考虑 空间 所以代价函数可以是线性的（查询处理view的开销等于view的size


4 Optimize DataCude Lattice

4.1 The Greedy Algorithm
傻逼一样的选cost最大的 view,而且空间限制的似乎是个数，而不是实际视图的存储空间；

cost 是什麽的cost 也没讲清楚；