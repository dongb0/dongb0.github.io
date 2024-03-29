---
layout: post
title: "[B+ Tree] 3 - B+树并发"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - data structure
---

我们知道数据库的事务通过一系列并发控制手段能够保证并发执行的事务操作数据时不会互相干扰，而索引作为事务执行时需要经常访问的数据结构，我们也需要考虑使用并发控制措施来保证多线程同步访问索引结构时不会得到错误的数据。

回忆一下我们在事务里用了什么机制保证事务并发执行的隔离性的：两阶段锁和基于时间戳的悲观并发控制机制，还有基于有效性检查的乐观并发控制，还有MVCC机制。这些并发机制为了确保事务的串行化而加入了许多要求，如两阶段锁的增长阶段和收缩阶段；而索引对可串行化并没有要求，一个事务执行期间，索引结构发生改变对事务并不会产生影响。比如事务T1第一次访问 key 15，对应的 value 存放在 slot 10,第二次访问时由于其他事务修改了索引，该 value 位置变为 slot 20,但事务T1依然能够读取到它想要的数据。所以索引的并发机制不需要像事务那么复杂，只要能保证数据的正确性即可。

教材[^1]中提到两种并发访问 B+ 树的技术，我们这里主要介绍其中一种： crabbing protocal，教材译为蟹行协议。

该协议要求我们按照以下规则对 B+ 树的节点加锁：

- **查找**时，从根节点开始获取共享锁，然后尝试申请子节点的共享锁，成功之后再释放父节点的锁，直到获取叶节点的共享锁。
- **插入或删除**时，从根节点开始申请互斥锁，然后申请子节点的互斥锁，成功获取子节点的互斥锁之后，检查子节点是否安全（插入时安全指不会分裂，删除时安全指不会underflow），如果安全则可释放子节点全部祖先的互斥锁（不包括该节点的锁）。为了这么做我们通常需要维护一个当前持有的互斥锁的队列。

以上的 crabbing 协议在对 B+ 树进行修改时可能会持有较多的互斥锁，考虑根节点的竞争程度比较激烈，这可能会严重降低并发程度（主要是读操作）。因此我们还有一个改进的 crabbing 协议：

- 查找操作与 basic crabbing 一样
- 插入或删除时，先按照查找的过程逐步申请共享锁一路向下走到叶节点，申请叶节点的互斥锁，然后检查叶节点是否安全；如果叶节点不安全则先释放手上持有的锁，然后退回到 basic crabbing 的插入/删除步骤重新从头逐步申请互斥锁；如果安全则可开始在叶节点上进行修改。

这一优化思路是尽量减少持有高层节点互斥锁的时间，这样如果某个修改操作不需要对树的结构进行调整，那么它在向下 crabbing 的过程中，其他的读写事务也能够访问索引（因为申请的是共享锁），而 basic crabbing 使用互斥锁锁住 root 或前几层节点时，其他所有事务都需要阻塞等待它释放互斥锁，可想而知在索引较大、并发事务较多时，这会严重降低并发度。

当然如果最后发现修改操作还是要调整树的结构，退回 basic crabbing 的步骤，那么我们申请共享锁的开销就完全变成 overhead。但是在统计后发现使用优化 crabbing 比起 basic crabbing 还是能够增加吞吐量，应当是实际场景中需要对树结构进行调整的操作不多，优化 crabbing 使用减少互斥锁的占用时间带来增益大于重新申请锁的 overhead 的缘故。

#### 实现注意事项

Bustub里B+树索引并发的并发基本上就是很直白的按照上述步骤实现，我觉得有一点额外要注意的地方是我们B+树中有一个变量 root_page_id_ 记录根节点的 page id，用于刚启动时从磁盘中载入 root page，这一变量是需要被持久化到磁盘上的（Bustub预留了 page 0 的位置存放它以及其他元数据），每次改变 root page 之后需要同步更新该变量。

但是在并发情况下，我们不能通过简单的互斥锁或者读写锁将它包起来。设想一下如果直接一把互斥锁在整个insert/delete过程中全程将它锁住，那么其他事务就无法通过 root page id 访问索引，我们前面做了那么多用来增加并发程度的努力就白费了；用读写锁情况也是一样，因为我们在insert/delete过程中都有可能修改 root page id，所以问题的关键在于确定我们**什么时候能够断定 root page id 不会再修改**。

问题的答案很明显：当root page在某次insert/delete过程中“安全”了的时候，它的page id当然也就不会再被修改。如何判断节点是否安全可以根据上面 crabbing 的过程再推一遍，我们最后可以简化为：只要我们准备释放 root page 的 latch（提醒一下这是用来保护 page 的锁，别跟我们这里保护 root_page_id 的互斥锁弄混了），这就意味着 root page 已经安全了。

这里简述其中一种实现方式：我们在crabbing过程中持有一个队列用于保存从根节点往下的latch的（队列内容实际是相应的page指针），只要稍微对它进行修改，在root节点加入队列之前，我们向其中先放入一个空指针用于特殊表示我们对 root page id 上了互斥锁。一旦 crabbing 过程中我们能判断某个节点是安全的，开始按照FIFO顺序释放它的祖先的锁，我们就能在释放 root page 的 latch 同时也根据这个空指针解除 root page id 的锁（能从队列中取出空指针意味着root page id也安全了）。这样就能满足我们的要求。

另外每次释放祖先的锁时，root肯定是第一个，我们也可以用一个变量表示这是第一次释放锁（也就是root），然后也同时解除 root page id 的锁。但是这种做法要么需要添加全局变量要么得在函数之间增加传递的参数，显然不如上面这种优雅。

#### 写在最后

关于 B+ 树并发算法的原理暂且就这么多，到这里B+树索引的介绍也就告一段落了。我们在 Bustub 中完成了一个 toy 数据库的实现（这里还只是涉及了其中的一种索引），功能和性能与工业界应用还有很大差距（人家动辄几十万行的代码可不是闹着玩的），不过起码我们迈出了理解数据库设计和实现的第一步。希望未来还能够像现在这样，还会因为理解和实现了一个曾经以为离我很遥远的东西而感到欣喜吧。


The End

---------------

[^1]: [数据库系统概念第六版 15.10]()