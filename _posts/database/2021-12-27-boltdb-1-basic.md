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

BoltDB是golang实现的一个kv-store，据说etcd的底层存储用的是它。这里打算做个简短的系列从源码层面学习一下事务的实现。说起来在这之前我甚至都不知道什么叫嵌入式数据库，后来搜了一些博客大概才知道像BoltDB、LevelDB这类，以一个库的形式被应用程序调用、用于在程序内部存储数据的，被称为嵌入式数据库（也许更多被叫做存储引擎）；像MySQL、Postgres这类独立运行在应用程序之外、通常需要通过网络传输数据库的关系型数据库（或者其他NoSQL），叫做数据库服务器。

> // TODO：还见过H2数据库（用来调试，是否还有别的用途？）

大概将会归纳分析以下几个功能模块的实现：

1. 事务

2. B+树索引

3. freeList

4. mmap的使用和原理

boltDB没有自己实现内存管理，而是直接采用mMap让操作系统来管理(所以还要看一下mMap的原理和使用方法)

我们这里关注boltDB如何保证事务ACID性质，阅读源码时参照了这篇[博客][1]。粗略来说 boltDB 采取了比较简单粗暴的实现方式，使得自身比较容易理解和实现（核心代码三四千行，加上所有适配不同平台的代码和测试不过1万行上下），但是也损失了性能和对更丰富特性的支持，我们可以通过与 CMU 15445 2020年的 bustub 对比来初步探查 boltDB 的独特之处在哪。

- 原子性
  - boltDB：使用shadow-paging实现，要对某页做写操作时，首先将其内容在内存中复制一份拷贝，然后在拷贝上进行修改；完成后再将新页持久化到硬盘中。
  - bustub：之前的 bustub 是通过日志来保证原子性的，在修改数据之前先将对应操作的日志持久化，以便发生故障后能够通过redo/undo来保证事物的all or none，2020年的实验未包括故障恢复部分。

- 一致性
  - 其他三个性质来保证一致性。

- 隔离性
  - boltDB的只读事务是不加锁的；写事务是通过单个线程来串行执行写入操作来解决并发修改的问题，修改得到新的数据页信息会写入meta数据页，使得后续的读事务能够看到本次写事务的修改；
  - bustub是通过读写锁的方式保证隔离性，每个读操作向 LockManager 申请共享锁，写操作申请互斥锁，LockManager 将锁的申请放入一个队列里来保证不会出现饥饿的情况；bustub中可以有多个读事务并发执行，而写事务允许一定程度的并发，只要能够申请到数据的互斥锁即可（如果两个写事务都申请同一个数据的互斥锁，那么它们会在该数据上排队，但是在此之前其他没有冲突的修改操作能够进行，比 boltDB 有更高的并发度）。

- 持久性
  - 二者都会在提交事务时开始将内存中的修改持久化到硬盘，持久化完成后事务才提交成功； boltDB 使用 WAL 也许可以在日志持久化之后就返回提交成功，目前因没有做日志恢复所以没有更多了解。


> Q: 读事务申请了一个mmapLock，写事务为什么没看到？  
A: 实际上读事务申请的是RLock，写事务会在需要mmap重新映射时申请一个WLock；都是符合逻辑的，读代码时不够仔细没看到罢了。

------------------


有一说一跟badger的接口还挺像

```
// boltdb read-write txn example
db.Update(func(tx *bolt.Tx) error {
	b := tx.Bucket([]byte("MyBucket"))  // Bucket retrieves a nested bucket by name. Returns nil if the bucket does not exist.
	err := b.Put([]byte("answer"), []byte("42"))
	return err
})

// boltdb read-only txn example
db.View(func(tx *bolt.Tx) error {
	b := tx.Bucket([]byte("MyBucket"))
	v := b.Get([]byte("answer"))
	fmt.Printf("The answer is: %s\n", v)
	return nil
})
```

```
// badger read-write txn
err := db.Update(func(txn *badger.Txn) error {
  e := badger.NewEntry([]byte("answer"), []byte("42"))
  err := txn.SetEntry(e)
  return err
})

// badger read-only txn
err := db.View(func(txn *badger.Txn) error {
  item, err := txn.Get([]byte("answer"))
  handle(err)

  var valNot, valCopy []byte
  err := item.Value(func(val []byte) error {
    // This func with val would only be called if item.Value encounters no error.

    // Accessing val here is valid.
    fmt.Printf("The answer is: %s\n", val)

    // Copying or parsing val is valid.
    valCopy = append([]byte{}, val...)

    // Assigning val slice to another variable is NOT OK.
    valNot = val // Do not do this.
    return nil
  })
  handle(err)

  // DO NOT access val here. It is the most common cause of bugs.
  fmt.Printf("NEVER do this. %s\n", valNot)

  // You must copy it to use it outside item.Value(...).
  fmt.Printf("The answer is: %s\n", valCopy)

  // Alternatively, you could also use item.ValueCopy().
  valCopy, err = item.ValueCopy(nil)
  handle(err)
  fmt.Printf("The answer is: %s\n", valCopy)

  return nil
})

```
-------------------


End

[1]: https://mrcroxx.github.io/posts/code-reading/boltdb-made-simple/4-transaction/


