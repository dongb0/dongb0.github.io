---
layout: post
title: ""
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - database
---

note 1：equitas 是 set semantic / 而 spes 是 bag semantic

note 1.1：Under set semantics, two queries are semantically equivalent if
and only if for all valid input tuples, the output tuples obtained after
executing the queries on the input tuples and eliminating duplicates
are equivalent


abstract
代数检查query equivalence的方法存在局限，难以检测SQL的语义（复杂谓词，三值表达式），且computationally intensive

因此本文提出一种 symbolic representation 的检测方法，思想是将query转化为 first order logic formulation，然后用 satisfiability modulo 理论来验证他们的等价性。

比起基于代数的方法，缩短了27倍的运行时间，并且在真实数据集中成功检测出其中11%的相似查询

1 Intro


sec 2 Algebriac approaches 的局限
sec 3 介绍representation-based 方法；rule-based extension for outer join（不看


2 Background

通过例子解释 代数方法 有几种等价无法检测；这里不重复

2.2 Symbolic Representation-based Approach


大概理解下来，是通过转化为 一些（？什么样）的表达式，然后通过SMT solver来判断这些表达式是否可以满足，来解决（所以这里用了Z3？

3 SPJ queries
证明等价，其实是证明两个查询互相包含；
而包含则是通过 SMT solver 证明
1） cond1 ^ 非 cond2 不可能满足
2） cond1 ^ cond2 ^ 非(cols1 = cols2) 不可能满足
来证明cond1 满足时 cond2 也满足，
以及cols1 和 cols2 等价
从而证明包含

3.3 Symbolic Representation Construction
这一节介绍递归构建 symbolic representation 的方法


3.4 Encoding Expr and Predicates

看不懂，不知道做了什么；

3.4.3 case constructor 就不看了

6 Evaluation

6.1 Implemetation
简单介绍equitas的流程图

6.2 Comparison against UDP
对比UDP的检测能力（在某个不认识的数据集

6.3 Efficacy on Production SQL Queries
在Ant 。。。 数据集上，17000个查询分为五组，检测出其中2000左右的查询存在 common sub-query。（但是怎么检测的？有接口吗？