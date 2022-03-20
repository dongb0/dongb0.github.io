---
layout: post
title: "[db/BoltDB] 4 - FreeList"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - database
  - golang
  - source code
---

上回我们说到，BoltDB 中删除元素之后，得到的空闲页面并不会归还给操作系统，因此它的数据库文件只增不减。但是从逻辑上来说我们肯定不能白占着资源不干活，BoltDB 也不会一直闲置这些空闲页面，而是通过 freeList 跟踪这些页面的位置，下次需要分配空间时优先使用 freeList 中的空闲页；如果列表中没有空闲页，再考虑扩容 mmap 获取更大空间（造成文件大小增长）。

// TODO：使用 freeList 的示意图
// txn1: Hey, I need 3 pages, OK, freelist grants it
// txn2: I need 2 pages, No free pages, resize mmap

简单看下 freeList 的定义：
```
type freelist struct {
	freelistType   FreelistType                // freelist type
	ids            []pgid                      // all free and available free page ids.
	allocs         map[pgid]txid               // mapping of txid that allocated a pgid.
	pending        map[txid]*txPending         // mapping of soon-to-be free page ids by tx.
	cache          map[pgid]bool               // fast lookup of all free and pending page ids.
	freemaps       map[uint64]pidSet           // key is the size of continuous pages(span), value is a set which contains the starting pgids of same size
	forwardMap     map[pgid]uint64             // key is start pgid, value is its span size
	backwardMap    map[pgid]uint64             // key is end pgid, value is its span size
  ...
}
```

### freeList 回收和分配页面

#### 分配页面

首先 freeList 肯定要提供回收页面和分配页面两个基础操作。freeList 由底层存储有 array 和 hashmap 两种实现，这里我们只看 hashmap 的版本。

DB 需要新页面时，会先调用 freeList.allocate 函数尝试从 freeList 中获取空闲页面。它的 hashmap 版本实现如下：
```
// hashmapAllocate serves the same purpose as arrayAllocate, but use hashmap as backend
func (f *freelist) hashmapAllocate(txid txid, n int) pgid {
	if n == 0 {
		return 0
	}

	// if we have a exact size match just return short path
	if bm, ok := f.freemaps[uint64(n)]; ok {
		for pid := range bm {
			// remove the span
			f.delSpan(pid, uint64(n))

			f.allocs[pid] = txid

			for i := pgid(0); i < pgid(n); i++ {
				delete(f.cache, pid+i)
			}
			return pid
		}
	}

	// lookup the map to find larger span
	for size, bm := range f.freemaps {
		if size < uint64(n) {
			continue
		}

		for pid := range bm {
			// remove the initial
			f.delSpan(pid, size)

			f.allocs[pid] = txid

			remain := size - uint64(n)

			// add remain span
			f.addSpan(pid+pgid(n), remain)

			for i := pgid(0); i < pgid(n); i++ {
				delete(f.cache, pid+i)
			}
			return pid
		}
	}

	return 0
}
```

这里有一个哈希表 freeList.freemaps 记录空闲页面的大小和位置，大概的映射是 \<continue free page size, set\<start page id>>，也就是 freemaps 的 key 是构成这段连续空闲空间的 page 个数， value 是这么大的空闲空间起点 page id 的集合。大致如下图：

// TODO：示意图

要分配 n 个页面大小的空间时，就先从 freelist.freemaps 中找有没有大小恰好合适的页面，有则直接返回；如果没有的话就遍历整个 freeList 尝试找一个更大的空闲空间，从中腾出 n 个页面分配出去；如果还没有就返回 0 表示 freeList 无法分配。

需要从 freeList 分配页面的情况比较简单，只有在写事务发生插入/修改/删除操作时才会出现，其中删除操作因为也对 node 进行了修改，这需要在 shadow page 上进行，所以也需要分配新页面。

#### 回收页面

回收页面的过程分为 pending 和 release 两步，首先在事务执行过程中产生的空闲页面，调用 freeList.free 放入 pending list 中，表示页面将被释放，但此时分配新页面还不会使用这些 pending page；接下来调用 release 函数才是把 pending page 真正标为空闲页面。BoltDB 有些独特的是，会每次写事务开始时调用一次 release 来回收上次事务产生的 pending pages，将它们与之前的空闲页面合并（如果可能的话）。

// TODO：为了什么？为什么不是每次事务结束时回收？

一个页面会在以下几种情况出现时被回收：

1. 写事务需要对 page A 进行插入修改，复制 A 得到 page B 然后在 B 上修改，此时 page A 可被回收，这种情况主要出现在 spill 过程中；
2. 写事务删除 Bucket 或 node 时（删除 node 主要出现在删除元素导致的 rebalance），空页面需要被回收；

总结来说就是 shadow page 被写入磁盘后，对应的旧页面就可以回收了。代码实现比较直白就不贴了。


### freeList 的持久化

既然 freeList 记录的是数据库文件中空闲页面位置这样的持久化信息，那么它本身当然也需要是持久化的，不能只存放在内存里。BoltDB 打开数据库时，从 meta 指向的页面读取 freeList，写事务在修改 freeList 之后也需要把 freeList 写到新的页面中。

由以下的 freeList.read / freeList.write 函数完成。

```

// read initializes the freelist from a freelist page.
func (f *freelist) read(p *page) {
	if (p.flags & freelistPageFlag) == 0 {
		panic(fmt.Sprintf("invalid freelist page: %d, page type is %s", p.id, p.typ()))
	}
	// If the page.count is at the max uint16 value (64k) then it's considered
	// an overflow and the size of the freelist is stored as the first element.
	var idx, count = 0, int(p.count)
	if count == 0xFFFF {
		idx = 1
		c := *(*pgid)(unsafeAdd(unsafe.Pointer(p), unsafe.Sizeof(*p)))
		count = int(c)
		if count < 0 {
			panic(fmt.Sprintf("leading element count %d overflows int", c))
		}
	}

	// Copy the list of page ids from the freelist.
	if count == 0 {
		f.ids = nil
	} else {
		var ids []pgid
		data := unsafeIndex(unsafe.Pointer(p), unsafe.Sizeof(*p), unsafe.Sizeof(ids[0]), idx)
		unsafeSlice(unsafe.Pointer(&ids), data, count)

		// copy the ids, so we don't modify on the freelist page directly
		idsCopy := make([]pgid, count)
		copy(idsCopy, ids)
		// Make sure they're sorted.
		sort.Sort(pgids(idsCopy))

		f.readIDs(idsCopy)
	}
}


// write writes the page ids onto a freelist page. All free and pending ids are
// saved to disk since in the event of a program crash, all pending ids will
// become free.
func (f *freelist) write(p *page) error {
	// Combine the old free pgids and pgids waiting on an open transaction.

	// Update the header flag.
	p.flags |= freelistPageFlag

	// The page.count can only hold up to 64k elements so if we overflow that
	// number then we handle it by putting the size in the first element.
	l := f.count()
	if l == 0 {
		p.count = uint16(l)
	} else if l < 0xFFFF {
		p.count = uint16(l)
		var ids []pgid
		data := unsafeAdd(unsafe.Pointer(p), unsafe.Sizeof(*p))
		unsafeSlice(unsafe.Pointer(&ids), data, l)
		f.copyall(ids)
	} else {
		p.count = 0xFFFF
		var ids []pgid
		data := unsafeAdd(unsafe.Pointer(p), unsafe.Sizeof(*p))
		unsafeSlice(unsafe.Pointer(&ids), data, l+1)
		ids[0] = pgid(l)
		f.copyall(ids[1:])
	}

	return nil
}
```
freeList 是 BoltDB 用于回收并重复使用空闲页面的辅助组件，也许是 shadow paging 机制才需要使用。但是我有些好奇为什么 BoltDB 不增加 Compact 功能用于整理文件中的空闲页面，在数据库空闲时把数据页都移动到文件前面，减小文件大小。也许对一个简单的数据库引擎来说这样的需求过于复杂了。

那么下次看哪个带有 Compact 机制的数据库呢。

The End

------------




