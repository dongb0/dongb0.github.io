---
layout: post
title: "CAP Theorem"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - paperReading
---

> 太长不看版：分布式系统最多只能满足CAP条件（Consistency, Availablity, Patition-Tolerance）中的两个。但是我还是想知其所以然，尝试用自己拙劣的阅读理解能力解读一下论文 [Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services](1) 对CAP理论的证明。


首先文章先定义CAP的具体含义。

1. Consistency(Atomic Data Object)

>  Under this consistency guarantee, there must exist a total order on all operations such that each operation looks as if it were completed at a single instant.

我的理解是：集群的每个节点，由于网络波动或者其他因素，他们收到客户端operation的顺序可能不一致，比如节点A收到的请求顺序可能是 abcdef，而节点B收到的可能是 acfedb。  

而Consistency要求我们设计的分布式算法能够以某种方式，对这些不同顺序的请求达成一致，然后所有节点都按照达成共识的请求序列来执行，使得集群所有节点上的执行结果都相同（彷佛整个集群就是一个节点一样）。

2. Available

> For a distributed system to be continuously available, every request received by a non-failing node in the system must result in a response. That is, any algorithm used by the service must eventually terminate.

这里定义的可用性似乎与现在一般常说的高可用性不太一样。不是要求系统一直可用，而是要求每一条发送给正常节点的请求，都会得到一个响应（哪怕是返回写入或者读取失败的信息）。

3. Partition Tolerance

分区容错指网络中的节点分为两个或者更多分区，分区之间节点的信息将全部丢失。


### 异步网络中的CAP定理

#### 定理

在异步网络（asynchronous network）中，无法同时满足可用性和一致性要求。

这里的异步网络指的是：

#### 证明

反证法

#### 结论

只能满足CA（不能容忍分区的出现）、CP（出现分区之后请求不能得到回应）、AP（出现分区之后，可能返回不一致的数据）。


### 部分同步网络中的CAP定理

#### 定理


#### 证明


### 一种在部分同步网络中的弱一致性算法

[1]: https://dl.acm.org/doi/pdf/10.1145/564585.564601