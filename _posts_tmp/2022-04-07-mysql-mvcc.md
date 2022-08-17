---
layout: post
title: "[理解MySQL] 1 - MVCC"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - algorithm
---

之前我们略微提过使用MVCC得到的隔离级别基本上来说自然就是快照隔离等级，也就是不会出现不可重复读、脏读和读未提交数据的情况。但是在 MySQL 中，使用 MVCC 依然能够把隔离等级降低到 Read_Committed，该隔离等级下允许不可重复读情况的出现。

这就跟之前对快照隔离的描述有些冲突了，如果快照隔离读的是事务刚开始时数据库的快照，那这份快照里根本就不会包含其他并发事务提交的修改，我们又如何能读到呢？

这与 MySQL InnoDB 实现 MVCC 的方式有关。

// To Be Continued