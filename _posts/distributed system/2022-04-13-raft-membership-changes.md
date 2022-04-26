---
layout: post
title: "[Raft] 3 - 集群成员变更"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - raft
---

这一节读了两次没完全搞懂 Joint Consensus 怎么使用 $ C_{old,new} $，然后搜了一些博客发现居然又提到每次增删只变更1个节点的单步成员变更，踏马搞得我更迷糊了；奈何被问到了，不得不看。后来发现单步成员变更原来是作者博士论文[^1]里提的，在 extended version 那篇里只涉及Joint Consensus（一种两阶段式的成员变更方法，只是因为 changes 分为两个阶段，跟2PC之类的并无关系）。行吧，一点一点来搞清楚。

### Single Server Approach

> 先明确一点：两种成员变更方法都是在线更新的，无需关闭整个集群重新配置集群信息；如果能接受 offline 更新那就不用这些复杂步骤了

虽然作者在博士论文里写道，因为单步变更实现更简单，且能通过多次单步变更来完成任意大小集群的配置，更推荐采用单步变更的方式。但是后来作者发现论文中描述的单步成员变更其实存在bug[^2],[^3]，需要加入一些额外机制。对我而言，我是先看了单步变更的原理才逐渐明白后面 Joint Censensus 是要解决什么问题。

首先来看如何进行单步成员变更（single-server membership changes）。

- 每次只能增/删1个节点，这也是为什么被称为单步节点变更；
- 进行节点成员增/删时，leader 首先将 conf 的变更（也就是集群成员的信息）写入日志，新配置记为 $C_{new}$，与此相对的旧配置记为$C_{old}$；收到该日志消息的节点可以立即应用新配置信息，不用等待日志提交；应用了配置信息意味着节点会看到集群成员信息的改变，如原有4个节点现变为3个节点，选举/共识时发送/等待信息的数量也需要改变（原本选举需要3票当选，现在需要2票；另外成员的地址也可能改变）
- leader 只有在一次变更完成之后（conf 变更日志提交）才能开始下一次成员变更

该过程的步骤基本就上述几点，还是非常简单的。这一过程不会干扰集群正常处理请求的逻辑，在发生成员变更过程中集群服务依然可用，论文 section 4.2 花了不少篇幅来阐述。我们这里主要关注为什么单步成员变更能够保证 raft 的正确性。

> 思考：成员变更过程会出现什么问题？

因为每个节点收到集群变更消息的时间无法统一，会出现一部分节点已经接受了新配置 $C_{new}$，而另一部分节点还没收到消息、依然根据$C_{old}$配置来安排行为逻辑的情况。这一过程会出现什么问题呢，extend version 论文[^4]中提到如果原3个节点的集群 $[1,2,3]$ 加入2个节点 $[4,5]$，现在某个时刻只有节点3 收到了新配置消息，此时$[3,4,5]$ 已经可以构成多数派选举出一个 leader 了（通常是3当选，因为4和5的日志还是空的）；但是对于节点 $[1,2]$ 来说，它们也认为自己还是 3 节点集群中的多数派（因为它们没收到消息并不知道集群要变为5个节点），它俩也能选出一个 leader。这就可能导致一个任期内同时出现 2 个 leader，破坏了 raft 一个任期最多一名 leader 的性质。   


![membership-change-problem](/img/in-post/post-raft-mambership/arbitrary-nodes-problem.png)


那么单步成员变更如何解决这一问题呢，论文中把这一问题归结为一次变更多个成员时，可能出现两个不相交的多数派导致的，那么一个自然的解决思路是：只要保证两个多数派之间节点有重合不就好了（论文里这一说法花了我可多功夫来理解）。而只要每次增/删只改动一个节点，就可以满足这一条件。

这样说下来肯定看得是一头雾水，我们还是来看图吧。因为每次变更只涉及1个节点，那么根据多数派至少需要 $\lfloor \frac{n}{2} \rfloor + 1$ 个节点的要求，至少会有1个节点同时存在于 $C_{old}$ 和 $C_{old,new}$ 的多数派中，两个多数派再少一票都无法构成多数派了，所以这个节点的投票是决定 leader 在哪一派中出现的关键，它投给哪一派，leader 就会产生在哪边。因为节点一个任期只能投一票，所以也不会出现两个 leader 的情况。

![single-server-1](/img/in-post/post-raft-mambership/single-server-1.png)

任意奇数偶数个节点的情况都与此类似，如果出现更多关键节点（中间重合部分的节点），就更不会出现双主的情况了。偷个懒就不画图了，原作者论文里有更多介绍。


> $C_{old}$ 和 $C_{new}$ 节点的行为会有何不同？  
$C_{old}$ 不知道 $C_{new}$，请求投票和统计票数都不管 $C_{old}$；但 $C_{new}$ 节点是知道 $C_{old}$节点的，也会向$C_{old}$发送 voteRequest 消息并统计他们的票数。至于 $C_{old}$ 收到消息后是否会投票给 $C_{new}$ 节点，就目前我对整个框架的理解应该是可能会投的。

那么上述变更出现的bug是什么呢，简单概括是 leader 在成员变更中发生切换的话，可能造成已提交日志的丢失。解决方法是在原有基础上，限制 leader 需要提交至少一条属于自己任期的日志（一般来说是每个leader当选之后写入的noop日志），才能提交成员变更的日志。

// TODO：more description and solution for this bug

### Second Approach: Joint Consensus

Joint Consensus 过程稍微复杂一些，可以分为两个阶段：

1. leader 将 $C_{old,new}$ 配置的变更写到日志中，然后广播给整个集群；收到新配置的节点可以立即应用该配置；在该配置下日志的提交需要 $C_{old}$ 和 $C_{old,new}$ 中的多数派都同意才能完成

// TODO： 2/3票， 5/9票  
// TODO： 主要疑惑在这个同时存在两种配置时，如何进行票数统计

2. 当配置变更的日志被提交之后，因为此时只有 $C_{old,new}$ 的节点才有可能被选为新 leader（包含一条更新的配置变更日志），所以 leader 可以正式将 $C_{new}$ 配置广播给集群，然后等待配置被应用即可。

------

Ref

[^1]: [Consensus: Bridging Theory and Practice](https://web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf)
[^2]: [知乎 - Raft成员变更的工程实践](https://zhuanlan.zhihu.com/p/359206808)
[^3]: [bug in single-server membership changes - raft-dev@googlegroups.com](https://groups.google.com/g/raft-dev/c/t4xj6dJTP6E/m/d2D9LrWRza8J)
[^4]: [In search of an understanddable consensus algorithm](https://raft.github.io/raft.pdf)