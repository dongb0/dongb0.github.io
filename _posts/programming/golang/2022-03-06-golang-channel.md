---
layout: post
title: "[golang] 1 - channel"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - golang
  - programming
---

channel 是不同 goroutine 之间进行通信的主要方式（除此之外可以通过系统调用申请共享内存方式，但更推荐使用 channel），本篇我们简单归纳 channel 的设计原理。

只要几句话就可以概括 channel 的用法：goroutine 向 channel 中发送的数据会按照先进先出的顺序被读取出来，如果 channel 没有缓冲区或缓冲区满时发送方会阻塞等待数据被读取；如果 channel 中没有数据，接受方同样会阻塞等待数据到来。

// TODO： linux 管道

这样的工作方式让我想起 Linux 中的管道，如果我们自己来做这样的功能，很自然就想到用队列（FIFO），再通过锁保证队列的互斥访问，有数据进出队列时使用条件变量来通知读者/写者，似乎就可以得到一个简单的 channel。但这样的实现存在一个问题： channel 不仅保证数据先进先出，还要求先向 channel 发送数据的 goroutine 先得到发送的机会，先从 channel 读取数据的 goroutine 先读到数据[^1]。也就是说假如有 10 个 goroutine 按照 1～10 号的顺序向 channel 中写数据时，因缓冲区满被阻塞了，那么缓冲区出现空间时，这 10 个 goroutine 依然要按照 1～10 的顺序得到写数据的机会。而我们只使用条件变量唤醒睡眠的 goroutine 无法保证这一顺序，需要别的机制来支持。那么我们来看看 channel 底层数据结构 hchan 的定义，看看标准库的实现比我们拍脑袋得到的想法高明在何处。

update：继续看了golang条件变量 Cond 的源码分析之后，发现 Cond 其实对于等待的 goroutine 也是保证 FIFO 顺序的，不过实现原理都与 channel 一样维护一个等待的

// TODO：实现简单 channel -> 条件变量 golang sudog 队列。也许我们简易的 channel 确实能够实现一样的效果？

> 本文尽量少出现代码，只阐明设计思想即可，如果更关注代码实现细节，可以参阅[Go 程序设计与实现 - channel][1]，本文主要也参考自该章节

```
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters

	lock mutex
}

type waitq struct {
	first *sudog    // sudog represents a g in a wait list
	last  *sudog
}
```

`sudog` 中的 `g` 是 GMP 模型中的 g，就不在这里展开讲了，如果不了解就简单记为一个 goroutine 就好。这里我们看到 hchan 中确实有一个循环队列 `hchan.buf`，同时还有两个分别代表等待读/写的 goroutine 队列 `recvq/sendq` （实际实现为链表）。

### 发送

发送数据时有以下三种情况：

1. 直接发送：存在等待读数据的 goroutine，则将其从 `recvq` 中取出向其发送数据；将 g 标记为 Grunnable，下次调度时将运行该 goroutine；
2. 写入缓冲区：若当前没有等待读数据的 goroutine，但 channel 存在缓冲区且仍有空间，则向缓冲区写入数据。
3. 阻塞发送：以上两种情况不满足时，发送操作被阻塞，该 goroutine 的信息被存到 `sendq` 队列中，然后陷入沉睡。

### 接收

接收数据时也有对应的以下三种情况：

1. 直接接收：存在被阻塞的发送 goroutine 时，可以直接读取数据，如果不存在缓冲区，则直接读取该 goroutine 的数据并释放它；如果存在缓冲区，则读取缓冲区第一个数据，将 goroutine 的数据放到缓冲区队尾，然后释放 goroutine。
2. 从缓冲区读：不存在被阻塞的发送 goroutine 但是缓冲区中有数据时，从缓冲区中读取队首的数据；
3. 阻塞读取：缓冲区中也没有数据时，读取操作将被阻塞，goroutine 信息存到 `recvq`，陷入沉睡让出 cpu。

### 关闭管道

关闭一个 nil 或已经关闭的 channel 将产生 panic。正常关闭时要做的操作主要是释放 `waitq` 和 `sendq` 中的所有 goroutine，让它们恢复调度。从已关闭的 channel 读取数据将立即返回对应数据类型的零值，顺带一提 context 包就是利用 “已关闭的 channel 读取数据将立即返回对应数据类型的零值” 这一点来实现的。

因为 runtime 包的代码太抽象所以暂时没有去啃源码，只是根据其他的源码分析博客归纳了 channel 实现的原理和大概工作流程。如果有机会想看看里面的等待队列的实现，底层数据结构是否就是互斥锁+队列的实现方式（或者是更底层的semaphore之类？注：不是 sync 包里的 semaphore，而是 runtime/sema.go 中用于支持 Mutex 实现的那个）。

The End

----------------

[^1]: [Go 程序设计与实现 - channel][1]

[1]: https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/#641-%E8%AE%BE%E8%AE%A1%E5%8E%9F%E7%90%86