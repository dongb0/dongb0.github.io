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
---

// 如何搞 common subquery？

abstract
与 EQUITAS 同样基于symbolic representation,但支持bag semantic； 
// 说实话，对于bag semantic 有些不太懂。这里说是table 包含重复tuple？那 equitas ？
同时也更快（3倍

主要解决 bag semantic 下的QE问题

1 Introduction

equitas 是证明包含关系（但是是 set semantic），所以如果 Q1 中某个行在结果中出现了多次，但在Q2中之出现1此，虽然set semantic 下他们是等价的；但bag semantic 下是不等价的；
equitas 会认为 Q1 也被 Q2 包含，从而得到Q1 Q2 等价的判断。

而bag semantic 才是数据库实际场景中常见的，

2 Background
主要就是解决 equitas 不能扩展到 bag semantic 的问题
方法从证明两个查询互相包含，变为证明两个查询的输出之间存在一个完全相等的双射，来满足 bag semantic 的要求。

3 Overview

证明分两步：
1）证明cardinal equivalence
2）证明full equivalence

通过递归式的构造 QPSR（query pair symbolic representation
从底层的table scan 一直到最上层的 QPSR,然后在最上层 QPSR 上使用SMT solver证明完全等价，以此来证明这是个identity map,从而证明得到这是个完全的双射；


4 Query Representation

4.1 Syntax and semantics
SPES 将查询分为4类：
1）table query
2）SPJ query
3）aggregate query
4）union query
除了 table query 之外，剩下的query 都需要其他的subquery作为输入
join在 SPES 中表示为 union query，接收两个 subquery 作为输入 

4.2 Normalization Rules
用来提升 SPES 处理bag semantic 能力

UNF query 
Union Normal Form query

这是一种 SPJ subquery 的 Union 形式


5 Equivalence Verification

5.1 Equivalence Deifinition
5.2 Query Pair Symbolic Representation
似乎是在 equitas 的基础上，将两个query 的 SR 组成了 pair；
（好处是？ 

5.3 Construction of QPSR
// 怎么判断级数（ cardinality ）相等呢？
首选需要是相同的查询类型，然后
// 这里说通过 normalize 之后，子查询也会是相同类型的； 那么还得重复看一下 normalization 到底做了什么

5.4 Table Query
两个query 的table等价，就说明 cardinal eq（甚至可以说是 full eq）
QPSR：因为是相等的，所以构建一个就够了；
// 但是到底如何用 T-Schema 来构建 symbolic tuple？

5.5 SPJ Query
对于join，首先验证Q1 join操作两个table的子查询与 Q2对应 sub-query的cardinality相等，若相等，则join之后的output table cardinality也相等。那么就可以进行接下来的 Predicate/ASSIGN 验证和构造 QPSR 了。

如果验证通过，则用Q1 Q2 的 COLS1/2 来构造新的 COLS（投影后）,然后拼接COND，这就是 SPJ 的 QPSR

5.6 Aggregation Query
5.7 Union Query

7 Evaluation

7.1 Implementation

7.2 Comparitive Analysis

7.3 Efficacy on Production Queries
同样，找ant 数据集中的相似，但是这次没有进行物化的实验

7.4 Limitations
不完全，有部分语法不识别 
incomplete

----------------

