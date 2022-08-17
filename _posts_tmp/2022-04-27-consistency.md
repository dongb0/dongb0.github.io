---
layout: post
title: "TODO：一致性"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - distributed system
---

consistency 和 latency 之间的取舍： strong consistency means high latency; low latency result in weak consistency


最终一致性（Eventual Consistency）的思想是

1.将数据复制到整个集群，
2.集群每个节点先在本地进行 update，然后将自己的更新广播给其他节点，使其他节点也能得知这些更新的内容

“最终（Eventual）“的含义是指，哪怕某些节点的更新存在冲突，也会根据约定的规则解决冲突，最后在集群内得到一致的数据，所以称之为最终一致性。

// TODO：跟 strong consistency 相比： performance up

强一致性（Strong Consistency，aka. linearizability）

// TODO

能提供强一致性的服务，通常都面临着可用性和性能上的问题：为了保证强一致性，我们无论读写都要联系集群所有节点来确认数据是否一致，容易看出一旦某些节点故障或出现网络波动，服务的质量就会大受影响。


Single Copy Protocol： Linearizble

Epidemic Protocol： sequentially consistent

 C in CAP means linearizability

但直觉上 Epidemic Protocol 应该还是会有问题，如何解决冲突？会不会两个节点用同一个时间戳（逻辑）

// 时间戳还附带了 Pid，

chap 2 通过主要复习离散的一些数学工具，主要涉及关系、图、同构，定义了事件图（Event Graph）
chap 3 定义一致性模型，举例展示事件图，定义 Commit Point, Visibility 和 Arbitration 用于判断 linearizability 和 其他一致性

chap 4 给出Values/Operations 定义，随后介绍 sequential data type，是由一个 Values Set，一个初始状态和一个 Operation Function Set 构成的集合。

Quiescent Consistency： A weak form of eventual consistency，但是给出的定义我看不懂