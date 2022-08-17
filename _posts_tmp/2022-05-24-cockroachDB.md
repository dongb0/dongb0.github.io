---
layout: post
title: "[db/cockroachDB] 1 - Intro"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
catalog: true
tags:
  - database
---

### 0 序言

本打算检索看看 multi-raft 是从哪开始使用的，结果就查到了 CockroachDB。正好之前已经列在 TODO List 上了，就干脆看看它的架构和特点吧。系列还是首先从 // TODO： 开始？

// TODO: paper
[]: [CockroachDB: The Resilient Geo-Distributed SQL Database](https://dl.acm.org/doi/pdf/10.1145/3318464.3386134)

// TODO：blog
[]: [diff between TiDB & CockroachDB](https://www.jianshu.com/p/8d0a99e198fb)

### 1 什么是CockRoachDB

CockRoachDB 是一个开源的分布式


### 2 CockroachDB架构

- SQL
- Transactional KV
- Distribution
- Replication
- Storage ==> RocksDB

Replication: Raft

Range Group: 是否就是按照 Key Range 切分？大概什么样的结构？

Range-level lease: leader
==> *leaseholder* ：serve authoritive up-to-date reads or propose writes ==> sec 4

Membership changes & load balancing




### 3 事务


### 4 SQL


### 总结


---------------
