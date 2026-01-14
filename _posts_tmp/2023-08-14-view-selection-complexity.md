---
layout: post
title: "[]"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - 
---

Title：
On the Complexity of the View Selection Problem

-----------------------------------------------

abstract

Harinarayan等人把MV selection问题形式化为：
将查询根据权重构成一个偏序集合，选择一个组视图进行物化等价于，从该偏序集合中选择某个子集，从而使某个cost function最小。

这一问题是NP-hard，所以常用渐进算法或者启发式算法。

本文证明以下下界：
1）Harinarayan,等人的贪心启发算法，查询响应时间至少为 n/12 * 最优 for infinitely many n（那不就变成无穷了？啥意思？
2）如果P!=NP，那么任意的线性视图选择算法得到的solution至少为最优解的 $n^{1-\epsilon}$ (for every \epsilon > 0)
3）