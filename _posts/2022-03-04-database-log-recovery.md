---
layout: post
title: "[db] 3 - 日志恢复"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
catalog: true
tags:
  - database
---

> 日志恢复的笔记

### 概念[^1]

事务的原子性要求事务对数据库的修改要么能够全部成功应用，要么把已经修改的部分全部撤回，恢复到该事务执行之前的状态。但是如果在一个事务在执行过程中发生了系统崩溃，内存中的 dirty page 没有来得及写入磁盘，就可能造成数据库状态的不一致，比如教材中举的例子：A从账户中取出\\$50转到B的账户，而\\$50转出A的账户，还没写入B的账户时发生了故障。

日志恢复系统就是用于解决这一问题、保证事务的原子性的。一条日志包含以下几个字段[^1]:

1. 事务ID
2. 数据ID，比如要修改的Page ID和offset
3. old value
4. new value

可以简记为$<T_i, X_j, V_1, V_2>$，另外用$<T_i, start>$表示事务开始，$<T_i, commit>$表示事务提交，$<T_i, abort>$表示事务终止。

数据库系统开始恢复数据时，首先查看日志记录，然后重做所有已经提交的事务，撤销所有还未完成的事务。由于我们是先写日志然后再写数据库，所以如果一项修改被应用到数据库那么它肯定会出现在日志中，重做它虽然会带来额外开销但是能保证数据库状态的一致；反之，没有出现在日志中，那么它肯定也没有被应用到数据库，所以我们可以放心的撤销那些没有$<T_i, abort>$标识的事务。 // TODO：needs update


但是日志的长度会随着运行时间而不断增长，要保留所有的日志，并且每次崩溃后都要扫描整个日志来恢复数据肯定是不现实的。可以通过使用检查点来避免上述问题。可以在日志长度超过一定阈值时，向日志记录中写入一个检查点，并且将

1. 内存中所有日志
2. 内存中所有 dirty page

持久化到硬盘；检查点中还包含一个记录当前正在执行事务的列表。通过使用检查点，我们能够确定在检查点之前提交的事务，它们的修改在检查点之前、或者伴随着检查点的刷盘操作应用到数据库中，因此在恢复数据时只考虑从checkpoint到日志结尾部分的事务，重做其中已经提交的，撤销其中未完成或者终止的部分，就能保证数据库能恢复到崩溃发生前的统一状态；对于checkpoint中记录的尚未执行完毕的事务，可能需要回溯checkpoint之前的日志来对该事务进行撤销，不过一旦撤销操作完成之后，$<T_i, start>$之前的日志记录就可以被删掉了。


### 崩溃后的恢复过程


#### 使用日志重做事务

重做需要从最后一个检查点开始，扫描到日志结尾，重做日志中碰到的$<T_i, commit>和<T_j, abort>$。具体步骤如下：

1. 初始化 undo-list 为检查点中记录的事务列表；
2. 若碰到正常日志记录$<T_i, X_j, V_1, V_2>$或事务回滚产生的记录$<T_i, X_j, V_2>$，我们将重做该日志记录的修改，即将$V_2$的值赋给$X_j$；
3. 若碰到$<T_i, start>$则将事务$T_i$加入 undo-list；
4. 若碰到$<T_i, commit>或<T_i, abort>$则把事务$T_i$从 undo-list 中去掉。

当我们扫描到日志结尾，也就将所有在崩溃前已经完成的(包括commit和abort)的事务重做完毕，虽然这其中可能有些事务崩溃前已经被应用到数据库，但是我们无法确定到底哪些事务没有被应用，所以全部重做是最万无一失的方案。重做结束之后 undo-list 中将只包含那些在系统崩溃之前没有提交也没有回滚的事务，将被用于下一步的撤销操作。

#### 使用日志撤销事务

恢复的撤销阶段我们从日志末尾往前扫描，来回滚事务。具体步骤如下：

1. 遇到属于 undo-list 中事务的日志$<T_i, X_j, V_1, V_2>时，撤销该日志记录的操作，写入一条$<T_i, X_j, V_2>$的日志记录
2. 遇到属于 undo-list 中事务的$<T_i, start>$，就往日志中写入一个$<T_i, abort>$标识事务回滚完毕，并将事务从 undo-list 中删掉

上述操作进行到 undo-list 为空时，我们就完成了所有事务的回滚。

如果恢复过程再次发生崩溃，撤销过程可能把一些本来属于未完成的事务变为了abort事务，也就是需要重做的事务增加了，而需要回滚的事务减少了。

// TODO： fuzzy checkpoint

// TODO： 分布式事务

#### WAL

 WAL（Write Ahead Log） 机制是指在一次写操作被应用到数据库**之前**，需要写在日志系统中记录下这次写操作的修改并持久化到磁盘。更具体的说是*内存中的某个 dirty page 写入磁盘之前，所有与之相关的日志都必须先持久化到磁盘*。

### 延伸：InnoDB中的double write

> 其实这篇文章就是为了搞清楚什么是 partial write 以及 InnoDB 的 double write 机制才写的（因为搞不清楚为什么不能直接redo），属实是为了这点醋包的这顿饺子了。

在数据写入某个 page 的过程中发生宕机的话，可能造成部分写失效(partial write)，比如 16 KB 的页只写了前 4 KB，但是书中提到如果这个页本身已经发生了损坏，再对其进行重做是没有意义的[^2]。一开始我很疑惑，在我的理解里，日志里应该包含了这个页面修改的全部信息，为什么不能直接重做。

随后我在查找资料过程中找到一篇[博客2](2)，说到这可能与InnoDB日志结构、页面有关。由此我找到了另一篇感觉比较有说服力的[博客3](3)，提到日志的格式有逻辑日志(logical logging)、物理日志(physical logging)、物理逻辑日志(physiological logging)，其中物理日志的格式是记录所修改数据的实际物理地址，比如页99偏移1001处，修改4字节长度的数据为'aaaa'。这种格式的日志消耗空间比较大，但是确实可以直接用于恢复出现 partial write 数据；逻辑日志直接记录sql语句，redo的时候需要重做整个sql（//TODO：对于undo如何实现）；物理逻辑日志格式则依赖页面中 header 信息，比如页面中的slot 1、2、3分别位于偏移1001、1045、1078处。假如发生了 partial write，header 和具体数据的映射关系无法保证依然正确，因此不能直接通过日志来进行数据恢复。从15-445的[课件](5)里扒了张对比三种日志格式之间区别的示例，感兴趣的可以去看[完整课程](6)。

![logging-schema](/img/in-post/post-log-recovery/logging-schema.png)

我不确定 InnoDB 使用的是逻辑日志还是物理逻辑日志，但反正肯定不是物理日志，因此需要采用一些别的措施来解决这一问题，也就是下一节要介绍的 double write。

##### 附加：数据库和文件系统的读写单元
我们知道一个 Database block 大于操作系统的读写单元时，数据库 block 写入磁盘不能保证原子性，有可能导致 partial write 问题。[博客3](3)中对比了 DB block \ OS block \ IO Block \ sector 的概念，本人因无法区分 page、frame、block 而深感苦恼，故重写。
- sector  
  是机械硬盘中最小的读写单位，通常为 512 byte，SSD中为了兼容也有一个概念上的扇区(logical sector)，不过目前新标准的硬盘扇区大小为4KB
  // TODO: Advanced format
  // TODO：对于机械硬盘的历史发展需要补充了解，PS：加上SSD
  // TODO：开新坑介绍HDD和SSD
- Block[^3]  
  是操作系统进行文件IO的最小单位，一般NTFS的block size为 4KB，所以默认配置下1个 block 会包含 8 个 sector。
  // TODO：所以一个目录大小是4K？
  // TODO：NTFS和EXT4等文件系统格式？
- Page[^3]  
  是CPU从内存中读取数据的逻辑单元（// TODO：needs update），一般由一个或多个 Block 构成。
  // TODO： frame
- DB Page
  数据库读写数据的基本单位，InnoDB默认为16KB。

### 延伸2：如何避免 partial write 问题

[博客3](3)提到日志的 block 一般为 512KB，可以认为日志本身不会出现 partial write。InnoDB的 double write 机制则可以很好的解决数据页面的 partial write，大体流程如下：

内存中分配有 2MB 的 double write 缓冲区，对应128个 16KB 的 page；磁盘上也分配有连续的 2MB double write 共享表空间。每次刷盘时先把脏页存到内存中的缓冲区，缓冲满时先顺序写入 double write 的共享表空间中（我猜可能是满1MB时），这部分的写入完成后再正式将页面写到实际的存储页面。这样当数据页发生 partial write 时，总能从 double write 的共享表空间找到完整的页面数据。

但是还有一个问题，写备份的时候会不会出现 partial write 呢？应该也是有可能出现的，但是此时数据库页面还没有进行修改，对于出现部分写的 double write 页我们可以直接丢掉。

如果能够保证页面原子性的写入磁盘中，就不会出现 partial write。
有一些文件系统本身提供了防止 partial write 的机制，如 ZFS 文件系统[^2]


-------------

### Ref

[^1]: [数据库据系统概念-第16章]()
[^2]: [MySQL技术内幕-InnoDB存储引擎（第二版)2.6.2](1)
[^3]: [Difference between Page and Block in Operating System](4)

[1]: https://github.com/wususu/effective-resourses/blob/master/%E6%95%B0%E6%8D%AE%E5%BA%93/MySQL%E6%8A%80%E6%9C%AF%E5%86%85%E5%B9%95(InnoDB%E5%AD%98%E5%82%A8%E5%BC%95%E6%93%8E)%E7%AC%AC2%E7%89%88.pdf
[2]: https://www.percona.com/blog/2006/08/04/innodb-double-write/
[3]: https://blog.51cto.com/mengphilip/1672113
[4]: https://www.javatpoint.com/page-vs-block-in-operating-system
[5]: https://15445.courses.cs.cmu.edu/fall2020/slides/20-logging.pdf
[6]: https://15445.courses.cs.cmu.edu/fall2020/schedule.html
