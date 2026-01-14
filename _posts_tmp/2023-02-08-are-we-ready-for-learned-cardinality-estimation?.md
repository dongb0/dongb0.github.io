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

abstract
从三个方面探讨 基于学习 的基数估计；
1. 静态环境下，传统方法和基于学习方法的性能：此时基于学习的估计准确率更高，但需要训练
2. 动态环境下，基于学习的方法容易出现较大错误；变化较小时，准确率差距不大
3. 基于学习的方法出现错误的原因：出现关联（查询？）、偏斜的变化时，同时难以预测和解释为何出错；

结论：现在还无法应用在工业界，不够成熟



2 Learned Cardinality Estimation

2.1 Problem Statement

就是估计一个查询
$$
SELECT COUNT(*) FROM r
WHERE \theta_1 AND ... AND \theta_d
$$
返回结果中包含的 tuple 数目。

2.2 Taxonomy

看起来大概分为两类：
a）Regression Methods
b）Joint Distribution Methods


