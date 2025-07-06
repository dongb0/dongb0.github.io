---
layout: post
title: "[Linux] Page Structure"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - Linux
---

关于Linux内核，想了解几个部分：
1. 内存管理有关回写page、write back线程的部分。
2. 内核的page实现方式是什么样，是否有可以参考的地方。 

> 主要阅读资料是深入理解linux内核的15章。

内核中管理的页面放在struct page里，占60字节，用unsigned long flags中的bit来表示页面的各种状态。page可以归属于不同的owner，比如普通文件、从磁盘加载的数据、特殊文件系统的数据等。

page到底属于哪个owner，是通过page指向的inode+文件offset来判断。具体来说是inode结构体中有一个address_space对象，page内存储一个指向该对象的指针。
> 回忆inode是什么：inode实际是一个文件的元数据，包含了文件存储的数据块位置、长度、读写执行权限、用户/用户组等信息。

如果缓存中存储了大文件，为了加快查找过程，内核会为每个address_space构建一个Radix Tree。扇出64，直接用page index来填充。比如层高为1的树可以存储page index 0~63的页面，如果存储的页面不在此范围内，则树需要长高。比如现在存放了index 0和index 131的page，则树高需要为2；最高可以达到6层，对应4TB内存。

有了基数树之后，我们可以介绍在page cache中增删查改页面的基本流程。

- find page
  - find_get_page()调用radix tree的radix_tree_lookup()来搜索页面，需要先持有address_space的spin lock；，搜索完成后释放锁；如果找到返回page指针，没找到返回null。find_lock_page()则是在此基础上对page加锁（通过设置怕page的PG_locked标记），如果page已被所住，则将page放入队列中，等待其他线程调用unlock_page()之后唤醒。类似的还有find_trylock_page()不阻塞直接返回，如果page不存在则返回错误码。
- add page
  - 调用radix_tree_preload()关闭内核的抢占以及申请节点内存
  - 拿mapping->tree_lock锁
  - radix_tree_inser()插入节点
    - 如果新节点大于树当前的表示范围，需要增加树高
    - 从根节点往下遍历，并分配必要的中间节点
  - 设置page count和PG_locked标记，后者防止内核并发操作这个无效的page
  - 初始化page，释放address_space自旋锁
  - radix_tree_preload_end()，完成插入
- remove page
  - 申请树的锁
  - radix_tree_delete()删除节点，包括：
    - 从根节点往下遍历到目标节点，最后从叶节点开始清空node.children数组中对应的（page/node）指针，并且减少node count；如果count减到0则释放node内存。
    - 递归这一过程直到回到根节点
- update node
  - 调用find_get_page()获取page
  - 如果page不存在，则：
    - 调用alloc_page分配frame
    - 调用add_to_page_cache()将page插入page cache
    - 嗲用lru_cache_add()将page插入对应zone的LRU链表
  - 如果page存在，则调用mark_page_accessed()更新LRU链表
  - 如果page不是最新的（PG_update标记被清楚），则用给用的read函数从磁盘中page最新版本
  - 返回page指针

同时为了加快查找radix tree中所有dirty/writeback节点的流程，树的中间节点会记录这个子树下是否存在dirty/writeback节点，如此可以跳过那些完全不存在目标节点的子树。这个操作通过两个int64作为位图来完成，当然了，位图的维护在增删改节点时需要引入一点点额外开销。


下面再介绍page cache的刷脏和淘汰的一些机制。

#### Writing Dirty Pages to Disk
内核刷脏的时机：
- 脏页数量超过阈值（或page cache大小超过阈值）
- 页面变脏超过一定时间
- 应用程序主动触发刷盘（通过sync、fsync等方式）

#### pdflush kernel threads
pdflush 是 linux 内核2.6引入的内核线程池，其线程数量动态可调。内核会创建2~8个pdflush线程，如果某个线程空闲超过1s会被删除；入股哦所有pdflush线程都忙碌超过1s则会创建新的线程。

#### Looking for Dirty Pages To Be Flushed
每次扫描所有radix tree来找脏页的开销比较高，内核采取的策略是：每次有线程写入页面，造成脏页水位超过background_threshold之后，就执行background_writeout()函数进行刷脏，这个水位默认设置为内存的10%（配置在/proc/sys/vm/dirty_background_ratio文件中）。

1. 实际刷脏会把脏页刷到40%水位以下（配置在/proc/sys/vm/dirty_ratio文件中，我看了下我的开发机是20）
2. 调用writeback_inodes()会尝试刷1024个脏页
3. 如果没刷够目标值，或者某个页面被跳过了，说明可能IO拥塞，则线程休眠100ms之后再重试


### slab
// TODO：内存分配