---
layout: post
title: "[db/BoltDB] 3 - Indexing: B+ Tree"
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

上次我们在 [BoltDB Transaction][1] 中简略介绍了 BoltDB B+树索引的实现，但是有些细节没有具体介绍。本文就来结合代码详细归纳一下 BoltDB 如何用短短 400 多行代码实现 B+ 树的（加上Cursor.go约700行）。

首先我们看 B+ 树节点的定义，根据我的理解给每个字段加上了注释。

// TODO: code block one tab use 8 space in blog page now, prefer 4

```
// node represents an in-memory, deserialized page.
type node struct {
	bucket     *Bucket    // indicates this node belongs to which Bucket
	isLeaf     bool       // leaf node or branch node
	unbalanced bool       // if needs rebalance, only set true when deletion happends
	spilled    bool       // if node has beed spilled(write to new page)
	key        []byte     // seperate key for this node
	pgid       pgid       // page id
	parent     *node      // parent node pointer
	children   nodes      // nodes type is []*node, used in spill()
	inodes     inodes     // []*inode, branch node stores children's page id, leaf store k/v pair
}

type inode struct {
	flags uint32
	pgid  pgid
	key   []byte
	value []byte
}
```

接下来我们看 B+ 树的增删查改如何实现。

### 查找

BboltDB 为 B+ 树实现了一个迭代器 Cursor，提供查找和范围遍历操作。Cursor移动的过程不用加锁，因为读事务不会改变页面内容，而写事务如果因为写操作在 node 之间移动时，会沿着路径拷贝 node。以下是 Cursor 的定义：

```
type Cursor struct {
	bucket *Bucket
	stack  []elemRef
}

// elemRef represents a reference to an element on a given page/node.
type elemRef struct {
	page  *page
	node  *node
	index int		// cursor index in node/page's inode
}
```

Cursor 的结构很简单，只有一个 Bucket 指针和一个存放路径的 stack。再看用来查找的 Seek(key []byte) 函数实现:

```
func (c *Cursor) Seek(seek []byte) (key []byte, value []byte) {
	k, v, flags := c.seek(seek)

	// If we ended up after the last element of a page then move to the next one.
	if ref := &c.stack[len(c.stack)-1]; ref.index >= ref.count() {
		k, v, flags = c.next()
	}

	if k == nil {
		return nil, nil
	} else if (flags & uint32(bucketLeafFlag)) != 0 {
		return k, nil
	}
	return k, v
}
func (c *Cursor) seek(seek []byte) (key []byte, value []byte, flags uint32) {
	_assert(c.bucket.tx.db != nil, "tx closed")

	// Start from root page/node and traverse to correct page.
	c.stack = c.stack[:0]
	c.search(seek, c.bucket.root)

	// If this is a bucket then return a nil value.
	return c.keyValue()
}
```

Cursor.seek() 从根节点出发，把球传给 Cursor.search()。
Cursor.search() 函数首先通过 page id 从 Bucket 的缓存中获取 node（node不存在时从 page 解析得到；如果是读操作则直接从 mmap 中读取并返回 page），如果还没到达 leaf，则通过 searchNode()/searchPage() 继续递归寻找，二者只是为了处理不同的数据类型，函数逻辑基本一致，因此这里只展示其中一个。到达 leaf 之后，使用二分查找来搜寻我们需要的 k/v pair 并返回，查找过程到这就结束了。

```
func (c *Cursor) search(key []byte, pgid pgid) {
	p, n := c.bucket.pageNode(pgid)
	...
	e := elemRef{page: p, node: n}
	c.stack = append(c.stack, e)

	// If we're on a leaf page/node then find the specific node.
	if e.isLeaf() {
		c.nsearch(key)
		return
	}

	if n != nil {
		c.searchNode(key, n)
		return
	}
	c.searchPage(key, p)
}

func (c *Cursor) searchNode(key []byte, n *node) {
	var exact bool
	index := sort.Search(len(n.inodes), func(i int) bool {
		// sort.Search() finds the lowest index where f() != -1 but we need the highest index.
		ret := bytes.Compare(n.inodes[i].key, key)
		if ret == 0 {
			exact = true
		}
		return ret != -1
	})
	if !exact && index > 0 {
		index--
	}
	c.stack[len(c.stack)-1].index = index

	// Recursively search to the next page.
	c.search(key, n.inodes[index].pgid)
}
```

但是需要范围遍历时，因为这里的实现没有在叶节点中保存指向右边叶节点的 next 指针，所以当遍历走完一个叶节点之后，需要先通过 stack 返回父节点，随后移动 elemRef 中的 index 指向下一个元素，再调用 Cursor.first() 沿着该元素指向的页面访问它叶节点的一个元素。

个人认为，之所以需要这么处理是为了能够协调 node 和 page 两种数据格式，毕竟 page 无法保存 next 指针，也无法知道它下一个页面会是谁。如果 BoltDB 设计 B+ 树为必须将 page 的原始数据解析为 node，那么应该可以使用 next 指针来访问下一个叶节点，更方便进行范围遍历。但是因为使用的是 mmap，我们无法保证解析出来的 node 一直驻留在内存中，所以要考虑将要使用的 node 被替换出内存写入文件的情况（成为 page），这应该就是使用把缓存的控制权交给操作系统而不是自己实现缓存管理的 trade off 吧。

### 修改

上一篇介绍事务时已经讲过了 B+ 树修改的过程，BoltDB 从根节点开始搜索，将 Cursor 移动到 Key 所在页面，最后对叶节点中的 k/v 进行修改。这一过程经过的路径都会变为 dirty page，需要先拷贝一份来在新页面上进行修改，然后把新页面写入磁盘，把旧页面放入 freeList 等待回收。freeList 的具体用法和源码解析将在下一篇文章中介绍。

// TODO： 看了 freeList 之后，修改 txn 里 事务的图，加上 freeList 

### 插入

创建 Bucket 时先创建一个空的 root node（isLeaf:true），这样插入第一个 k/v 和后续的插入操作逻辑上都统一了：只管移动 Cursor 到叶节点，然后往里插数据就是。另外要记得在插入过程中 node 不会像教材描述的 B+ 树一样增长到最大长度后就进行分裂，而是一口气全放在叶节点中；BoltDB 会在事务提交时才统一将超出阈值的 node 分裂成若干个，更新 parent_node.inode 中的指针（page id）指向新 node，如果 parent 也需要分裂则继续重复这一过程。

> 分裂操作在 spill() 过程中一并完成，参见[BoltDB Transaction][1]

阈值根据 Bucket.FillPercent 计算得到，设为 `pageSize * fillPercent`，默认 pageSize=4096, FillPercent=0.5，所以分裂之后每个 node 大约为 2KB，但是因为 k/v 长度不是定长，所以每个页面中包含的 k/v 数量不一定相等。

// TODO：示意图？ 直接搬运前一篇？

// TODO：另外对于特别大的 k/v（比如 key + value > 2KB），每个 page 中数据如何存放？ overflow？

### 删除

删除操作会在 node 中留下空隙，回忆一下教材上定义的 B+ 树删除元素之后要做什么：如果元素个数小于 minSize 则需要合并或者在兄弟节点间重新分配元素。但有趣的是 BoltDB 只要求叶节点最少持有1个值，内部节点最少持有2个指针（实际上是 page id），也就是说大部分删除操作都不需要 merge/redistribute ，但是如果把叶节点删空了，或者内部节点元素少于2个时，我们还是需要对树的结构进行调整。
```
func (n *node) minKeys() int {
	if n.isLeaf {
		return 1
	}
	return 2
}
```

这一过程在 BoltDB 中是通过 rebalance() 函数完成的。写事务在提交时，尝试对 Bucket 中每一个缓存的 node 进行 rebalance 操作，当然了，实际 rebalance 的工作还是由 node.rebalance() 来完成的。

> 记得我们说过，写事务中使用 Put/Delete 进行修改之后，会将修改的叶节点到根节点这一路径上的 node 缓存到 `Bucket.nodes map[int]*node` 字段中，事务提交时该字段内存放着本次事务所有修改的 node。

```
// rebalance attempts to balance all nodes.
func (b *Bucket) rebalance() {
	for _, n := range b.nodes {
		n.rebalance()
	}
	for _, child := range b.buckets {
		child.rebalance()
	}
}
```

rebalance 的代码稍微有点长，我们一点点来解析。首先通过 unbalanced 标记判断是否需要进行 rebalance，该标记只会在进行 delete 操作是被设为 true（Put 操作造成的溢出是 split 需要操心的事，rebalance 只负责清理 size 小于 minKey 的节点）；另外后面我们将看到 rebalance 操作是向 parent 递归执行的，因此也需要这一标记避免重复 rebalance。

然后判断节点是否真的需要 rebalance，BoltDB 认为只有 size 在 $ 0.25×pageSize $ 以下或者元素数量小于 minKey 的节点才需要 rebalance，否则一律打回避免做无用功。

随后是对根节点为中间节点，且只剩1个指针这种情况的特殊处理，其他情况都可并入后续的逻辑。

```
func (n *node) rebalance() {
  if !n.unbalanced {
  	return
  }
  n.unbalanced = false
  
  // Update statistics.
  n.bucket.tx.stats.Rebalance++
  
  // Ignore if node is above threshold (25%) and has enough keys.
  var threshold = n.bucket.tx.db.pageSize / 4
  if n.size() > threshold && len(n.inodes) > n.minKeys() {
  	return
  }
  
  // Root node has special handling.
  if n.parent == nil {
  	// If root node is a branch and only has one node then collapse it.
  	if !n.isLeaf && len(n.inodes) == 1 {
  		// Move root's child up.
  		child := n.bucket.node(n.inodes[0].pgid, n)
  		n.isLeaf = child.isLeaf
  		n.inodes = child.inodes[:]
  		n.children = child.children
  
  		// Reparent all child nodes being moved.	// why update inodes? inode represent what?
  		for _, inode := range n.inodes {
  			if child, ok := n.bucket.nodes[inode.pgid]; ok {
  				child.parent = n
  			}
  		}
  
  		// Remove old child.
  		child.parent = nil
  		delete(n.bucket.nodes, child.pgid)
  		child.free()
  	}
  
  	return
  }
  ...
}
```

接下来就是通常情况下的 rebalance 逻辑了。
1. 如果节点空了那么直接回收该页面，并且从 parent 中删除指向它的指针；
2. 如果节点还没空，根据 minKey 约束它的 parent node 此时至少还有2个子节点，且根据 `n.size() > threshold && len(n.inodes) > n.minKeys()` 条件检查，本节点此时要么 size 太小，要么元素数量小于 minKey = 2（走到这一步的只能是中间节点，叶节点在 step 1就被回收了），需要与 sibling 节点合并；
3. 接下来就是找到 sibling 节点，然后把 node 和 sibling 合并，把被移动的 children.parent 指针指向新的节点，回收被删除的页面；
4. 最后，因为本次合并操作造成了 parent 的元素数量减少，所以还要向上继续回溯，查看是否需要继续进行 rebalance。
```
func (n *node) rebalance() {
	...

	// If node has no keys then just remove it.
	if n.numChildren() == 0 {
		n.parent.del(n.key)
		n.parent.removeChild(n)
		delete(n.bucket.nodes, n.pgid)
		n.free()
		n.parent.rebalance()
		return
	}

	_assert(n.parent.numChildren() > 1, "parent must have at least 2 children")

	// Destination node is right sibling if idx == 0, otherwise left sibling.
	var target *node
	var useNextSibling = (n.parent.childIndex(n) == 0)
	if useNextSibling {
		target = n.nextSibling()
	} else {
		target = n.prevSibling()
	}

	// If both this node and the target node are too small then merge them.
	if useNextSibling {
		// Reparent all child nodes being moved.	// delete target node, adopt target's children
		for _, inode := range target.inodes {
			if child, ok := n.bucket.nodes[inode.pgid]; ok {
				child.parent.removeChild(child)
				child.parent = n
				child.parent.children = append(child.parent.children, child)
			}
		}

		// Copy over inodes from target and remove target.
		n.inodes = append(n.inodes, target.inodes...)
		n.parent.del(target.key)
		n.parent.removeChild(target)
		delete(n.bucket.nodes, target.pgid)
		target.free()
	} else {
		// Reparent all child nodes being moved. // delete current node, move my children to target
		for _, inode := range n.inodes {
			if child, ok := n.bucket.nodes[inode.pgid]; ok {
				child.parent.removeChild(child)
				child.parent = target
				child.parent.children = append(child.parent.children, child)
			}
		}

		// Copy over inodes to target and remove node.
		target.inodes = append(target.inodes, n.inodes...)
		n.parent.del(n.key)
		n.parent.removeChild(n)
		delete(n.bucket.nodes, n.pgid)
		n.free()
	}

	// Either this node or the target node was deleted from the parent so rebalance it.
	n.parent.rebalance()
}
```
原版 B+ 树在删除时如果出现 `node.size < minSize` ，则需要根据它和sibling元素数量之和来判断是要进行 merge 还是 redistribute：

- if node.size + sibling.size > maxSize   
	===> redistribute node.size = maxSize/2, sibling.size = remaining elements  
		因为此时 node.size = minSize - 1（我们每删一个元素就立刻进行 rebalance 操作，所以 node.size 不可能更小），minSize = Ceilling(maxSize / 2），而 sibling.size > minSize && sibling.size <= maxSize  
		所以 node.size + sibling.size <= 2 * maxSize - 1，必定能够平分到两个节点中且每个节点 size 不超过 maxSize

- if node.size + sibling.size <= maxSize   
	===> merge node and sibling

而 BoltDB 中并没有用到 redistribute 操作，我分析这一方面与实现中对 minKey 定义实在太宽松有关，BoltDB 允许中间节点最少只持有两个 key，叶节点最少持有1个 k/v，那么进行若干删除之后节点中大部分地方都是空的，而每个节点的 maxSize 根据 k/v 的大小可能会有十几到几十个元素（保持每个页面分裂后约占用2048字节），所以通常进行 merge 操作给页面加上一两个元素不会造成溢出；

但是奇怪的是 merge 操作不是把小的节点合并到大节点中啊，代码里没有判断 node size 的逻辑，反而是直接根据 node 的位置来进行合并的，如果 node 是第一个元素就把右边的节点合并到 node 中；如果 node 不是第一个元素就把 node 合并到左边的节点中。这样虽然大部分情况下会把 size 比较小的当前 node 合并到左 sibling，但也可能出现 node 是第一个元素、且右边节点元素很多的情况，那就要移动很多子节点。不过实现逻辑确实简化了，比我之前做的 B+ 树把合并操作分为四种情况考虑来得更加优雅。

当然，凡事不可能完美，这样简化代码逻辑相应的也有它的代价。我能想到最为突出的缺陷就是页面太空，造成树的高度比半满的 B+ 树要高，查询性能必然有所下降。

如果非常不凑巧，合并之后还是发生了溢出，那么 BoltDB 还有补救措施。没错，又是 spill() 函数出场的时候了。看看 Commit() 代码中，rebalance() 结束之后紧接着就是 spill()，于是这里又要重复一次那段话了：“spill 将 B+ 树中发生修改的 node 导出为 dirty page 的同时，也将超过 threshold 的节点分裂为合适的大小”。所以我们最后还是会得到合适大小的 node。
```
func (tx *Tx) Commit() error {
  ...
  
  // Rebalance nodes which have had deletions.
  var startTime = time.Now()
  tx.root.rebalance()
  if tx.stats.Rebalance > 0 {
  	tx.stats.RebalanceTime += time.Since(startTime)
  }
  
  // spill data onto dirty pages.
  startTime = time.Now()
  if err := tx.root.spill(); err != nil {
  	tx.rollback()
  	return err
  }

  ...
  
  // Finalize the transaction.
  tx.close()
  
  return nil
}
```

// 删除 rebalance 示意图：

那么有关 BoltDB 中 B+ 树实现的介绍基本上就这么多了，还真是花了不小的篇幅。这部分代码整体上还是比较简洁的，可读性不错，除了结构体定义部分的注释有些迷糊，看完也不知道某个字段的作用。看下来对我而言比较新鲜的点有：1).延迟节点的分裂; 2).使用比较小的 minSize，3).以及 merge 过程左右节点的合并简化为两种情况的写法。

最后还有一点要提的是，BoltDB 的删除操作虽然把 k/v 从数据库中删掉了，但是这个页面占用的物理空间并没有归还给系统，也就是说 BoltDB 的数据库文件大小是不会减小的（哪怕对所有 k/v 全部执行 delelte，文件也不会变小，这点我验证过），这些空间将被纳入 freeList 中，下次再需要分配空间时，BoltDB 会先从 freeList 中取。那么更详细的介绍，我们就留到下次再说吧。


The End

-------------

[1]: /2021/12/28/boltdb-2-txn