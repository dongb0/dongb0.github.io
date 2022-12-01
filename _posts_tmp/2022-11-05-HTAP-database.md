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

// 看看 HTAP 的现状和发展趋势

abstract

comprehensive survey of HTAP;


1 Introduction
background 概括：现在的 HTAP 系统当然比分离的 OLTP/OLAP 更具优势，能支持上层系统作出更及时的决策，如根据销售商品的趋势变化，及时调整调度库存和广告策略；

HTAP 定义：粗略概括来说，是能够允许 TP 和 AP 在同一个 data store 之中进行；而无需 ETL 从 TP 系统中提取数据。 // 更准确定义查看 Gartner \[35,15]

Motivation: 过去出现的一些 HTAP 系统\[18–22, 29, 31, 42, 44] 共同的特征是 采用了 行存和列存 来实现高质量的 HTAP; 这些系统的存储和处理策略 几乎完全不同，取决于他们面对的 application 优先考虑 TP 还是 AP；

因此要在 workload isolation 和 data freshness 之间作出取舍；


overview
1）TP techniques：a) MVCC + logging；b）2PC+Raft+logging

2）AP techniques：
  a)in-memory delta and column scan
  b）disk-based delta and column scan
  c）colunm scan

3) data synchronization techiquesL
  a)in-memory delta merge
  b)disk-based delta merge
  c)rebuild from primary row store

4）query optimization techniques：
  a)column selection based on workload
  b)hybrid row/column scan
  c)CPU/GPU acceleration

5）resource scheduling techniques：
  a) workload driven scheduling
  b) freshness-driven scheduling


HTAP benchmark

Challenges and open problems


2 HTAP database
