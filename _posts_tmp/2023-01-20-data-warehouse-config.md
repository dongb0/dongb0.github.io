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

// 切记：速看，挑要点概括
// 主要是找 基于查询重写的视图生成 framework

考虑视图维护开销，
输出是包含重写后的查询

但是似乎只是纯理论？实际工程如何做？也没给实验

abstract
针对视图选择问题，提出一种面向关系模型的理论框架，将视图选择转化为 state space optimization 问题，并提供穷举和启发式算法
// 虽然文中称为 DW Configuration Problem

1 Intro

1.1 
// 是考虑维护开销的
但是作为 DW Configuration Problem，他们要求：1）所有的查询都能通过 views 来回答。
以及，2）要求查询开销 和 维护开销，这两个要求是冲突的；（不过视图选择中也是同样

1.2 Contribution 

基于multiquery graph，将问题转化为 state space search
// 主要看下怎么构建图，怎么转化状态

2 Related Work
3 

这里定义的 cost，是包含执行开销和维护开销的

4 A State Space Search Based Algorithm


4.1 The States
构造出的 multiquery graph，边是用来表示 predicate 的；不是很直观

4.2 The Transitions
state transformation rules

selection edge cut

join edge cut

view merging

4.3 

算法就是直白的穷举，但是下一节 似乎会加入一些启发式算法，帮助搜索？// 看一下



// 缺点可以列举挺多的