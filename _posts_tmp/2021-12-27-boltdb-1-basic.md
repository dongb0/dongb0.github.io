---
layout: post
title: "[db/BoltDB] 1 - Introduction"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - database
  - source code
  - golang
---

> BoltDB是golang实现的一个kv数据库引擎，据说etcd的底层存储用它来完成的，这里打算做个简短的系列从源码层面学习一下事务的实现。说起来在这之前我甚至都不知道什么叫嵌入式数据库，后来搜了一些博客大概才知道像BoltDB、LevelDB这类能够作为一个库被应用程序直接调用的可以被称为嵌入式数据库（也许更多被叫做存储引擎）；像MySQL、Postgres这类独立运行在应用程序之外、通常需要通过网络传输的关系型数据库，叫做数据库服务器。

// TODO：还见过H2数据库（用来调试，是否还有别的用途）

大概将会归纳分析以下几个功能模块的实现：

1. 事务

2. B+树索引

3. freeList

4. mmap的使用和原理

记得boltDB没有自己实现 BufferPoolManager，而是直接采用mMap(所以还要看一下mMap的原理和使用方法)

（这是一种系统调用吗？）
mmap => memory mapped，是将文件映射到内存中直接进行操作的手段，由操作系统来管理页面的读入和写出。

操作系统课上学的由操作系统使用 lru/mru/fifo 等等各种页面替换机制，就是用在此处。


elemRef.index 字段含义, node.inodes 的下标


bucket.sequence 是版本号吗？？？

-------
tx.pages 是脏页
tx.allocate 在事务中分配脏页，调用 tx.db.allocate(tx.meta.txid, count) 按照写事务的 id 从 mmap 中分配空间（分配 count 页）
可能从 freelist 中取，如果 freelist 空间不够就 resize mmap来获取空间




通过 unsafe pointer 转化的结构体，为什么有些字段在结构体定义里看不见？


------------------


有一说一这跟badger还挺像的。（对比一下


    // code


-------------------


事务ACID，boltDB分别如何保证的。[参考](https://mrcroxx.github.io/posts/code-reading/boltdb-made-simple/4-transaction/)

跟我之前做CMU 15445的半成品bustub对比的话，

原子性：通过shadow-paging？bustub如何实现

一致性：

隔离性：
CMU 15-445的bustub是通过读写锁的方式实现隔离性，每个只读事务向LockManager申请share latch，写事务申请exclusive latch，LockManager将锁的申请放入一个队列里来保证不会出现饥饿的情况。bustub中可以有多个读事务并发执行，而写事务必须等待所有未完成的读或写事务提交之后，才能串行执行。


读事务对申请了一个mmapLock（为什么），写事务为什么不用？


