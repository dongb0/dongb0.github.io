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

似乎是使用多臂赌博机（老虎机）进行在线索引推荐，面向 ad hoc,analytical workload。

ad hoc 可以理解为倾斜比较严重的？
// 完全随机、无法预测的

虽然原理部分只有三页，但是本身对于 多臂赌博机 的原理就还不太懂，回去看看西瓜书再说吧


4 MAB for Online Index Selection


编码分为两部分：
1）基本都是要包含索引的位置信息
2）covering index？ + size + usage info


reward：
创建时间 + 查询执行时间？这部分怎么说

增益定义为：
full table scan time - with index）* index list {i}

如果优化器没有使用 index i 那么增益会为 0;

以及需要减去index的创建时间等等