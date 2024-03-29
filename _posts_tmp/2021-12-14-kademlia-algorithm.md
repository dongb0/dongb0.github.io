---
layout: post
title: "[p2p] Kademlia"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - algorithm
  - p2p
---

> 之前接触了go-libp2p这个p2p网络库，它底层维护的DHT采用Kademlia算法，这次就通过论文和代码开始探索一下p2p的世界


### 基本概念

#### DHT
  DHT，Distributed Hash Table，是p2p网络中的（重要组成部分）？

  ##### 为什么需要DHT？
  假设我们只知道周围节点的地址，

  本质上也还是一个哈系表，存储着网络中其他节点的地址。但是由于分布式网络的特性，可能会频繁的出现节点加入和离开的情况，DHT中很难保存所有节点的地址，那么如何利用有限的entry来尽量高效地获取其他节点的地址就是我们要考虑的一个问题。
  
  ##### DHT做了什么？
  前面我们提到过，一个节点不会保存整个网络内所有节点的地址，那么当我们需要与未知地址的节点通讯时该怎么办。思路很简单，向所有已知地址的节点发送

  这就是Kademlia算法解决的一个问题。

  与Chord算法相比，Kademlia计算节点之间距离时采用的两个key值之间的异或结果，这使得Kad能够从收到的消息里获取路由信息（how？

  ##### DHT与DNS对比

  DNS：中心化，当然也就伴随着中心化星型结构的{}等优点和{}等缺点，所有域名信息都存在域名服务器（至少在根域名服务器中）（修改阿里云里域名服务器的记录，会更改哪里的DNS记录？）
  DHT：分布式，

  Kademlia使用160 bit长度的key值

  #### 节点间的距离
  
  Kad将网络中的节点看作二叉树上的节点

  Kad定义两个节点x和y之间的距离为他们key值的XOR运算结果，即d(x, y) = XOR(x, y)

  这样一来可以利用以下的异或运算性质：

  结合律
  交换律
  三角不等式 d(x, y) xor d(y, z) >= d(x, z)
  a + b >= d(a, b)


  这样（哪样）两个节点之间的距离其实就是二叉树上两个节点的最近公共节点？（不准确

### 原理

#### 节点

每个节点维护一个\<Ip addr, UDP port, Node Id\>的三元组列表，（For each 0<= i <= 160, keep a list of <> tuple ? what does it mean?
一共k个桶（configurable

对于i比较小的情况，通常用不了k个桶，比如在树里比较高的几层？那么为什么会出现这样的节点呢？ Kad的key长度为160bit，那么还会出现0001,101等这种长度只有4和3的key值吗，如果有的话，补全到160bit那不还是会有一长串的0？

长度i的key就有2^i个？（整个例子） 以自己为视角拆分出若干子树，每个子树一个路由表，表里最多K项，

如果bucket满了，从桶里选最旧的地址ping一下，如果不通则用新节点地址替换；如果通了，就把旧地址移到队尾（表示最近用过），新节点的地址就丢弃掉不存了。
这样的设计是基于对「某个」数据集的分析，分析结果表明过去在线时间长的节点未来一小时内仍然在线的概率更高，比如保持在线1500分钟的节点在未来60分钟内在线的比率大概在90%左右；在线10分钟的节点未来60分钟内仍然在线的比例不到70%。（感兴趣的参见论文Fig.3)

(参考)[https://colobu.com/2018/03/26/distributed-hash-table/]



#### Kademlia协议

  Kad定义了四类消息：PING，STORE，FIND_NODE，FIND_VALUE
  PING：查看节点是否在线
  STORE：存储<key, value>
  FIND_NODE：使用160 bit的节点ID作为入参来寻找节点
  FIND_VALUE：同样接受节点ID作为入参，如果key对应的pair存在，则返回存储的value


  #### 节点发现
  
  给定一个节点如何定位最近的k个节点地址。

  首先从最近的桶里找 alpha 个节点地址（如果不够 alpha 则取最近 alpha个），然后向这些节点发送FIND_NODE RPC。获取到新的节点地址后，Kad从k个最近的地址中选 alpha 个发送FIND_NODE，如果这一轮节点发现没有找到距离目标更近的节点，那么从k个中选出还没查询过的 alpha个继续发起查询；如果k个查询仍没找到目标节点则终止递归，查找失败。

  STORE RPC的过程也类似。节点也会将STORE RPC发送给最近的k个节点进行存储，这些节点还会republish来保证存储内容有充足的备份。（细说

  FIND_VALUE只要有一个节点返回了相应的value，递归查询就立刻停止。同时出于缓存的目的，发起请求的节点在获取数据后还会发起STORE RPC将数据保存在那些自己查询路径中经过、但是没有存储这份数据的节点上。（这应该是博客上说的对热点数据的缓存）同时为了避免过度缓存，数据的有效时间将被设为（当前节点与？）之间节点数量指数的倒数，即距离越远有效时间越短。

  另外Kad考虑到如果某个bucket范围内没有有效路由表，但是又有存活节点的情况，每个小时从该bucket的地址范围内随机抽取一个节点地址尝试进行节点连接。


  (关于地址的问题)：哪怕是000......001，那也是最底层的叶子，关键在于用什么表示地址，160 bit 20字节，应该是用[]byte存，那么肯定有补全的，怎么在地址里区分 000000001 和 001 呢？ （为啥区分，这就是同一个地址，都是对应地址1），但是在二叉树上的位置是不是不一样？（理论上地址应该是需要补全的，所以每个地址最后都对应一个叶子节点，且应该都在同一层，

  ## libp2p里的实现是否是这样的？（要看这块！

  #### 路由表

  Kad的路由表相当于二叉树。
  （这块主要是路由表，或者说二叉树的构建过程）
  要解决树的不平衡问题

（这块搞不明白，算了）

#### republish

为什么需要republish：

1. 持有某段数据的节点可能下线
2. 新加入的节点没有内容

因此需要republish来保证数据能够存放在距离key值最近的k个节点中


