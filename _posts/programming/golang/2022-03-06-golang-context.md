---
layout: post
title: "[golang/std] 1 - context"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - golang
  - programming
  - source code
---

> 用golang有一段时间了，但是一直以来都是要用什么就查什么，时间一长啥也记不住。所以计划一点点归纳接触到的各种golang机制和标准库的用法，加深印象，也方便查阅。


在项目里接触过一些 context 的用法，之前只知道能够用来同步终止多个goroutine，一般来说我们使用 context 的方式如下面代码片段所示：
```
func foo(ctx Context, args interface) {
	select {
	case <- ctx.Done():
		return 
	default:
		// do something
	}
}
```

现在来扒一下源码看看标准库如何实现 context的功能。首先 context 接口有以下四个方法：
```
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key interface{}) interface{}
}

```
并且有四个用来创建 context 的方法：
```
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) 
func WithValue(parent Context, key, val interface{}) Context
```
可以看到`Done()`函数返回的是一个channel，实际上 context 的 cancel 操作就是通过关闭其内部的 channel 来完成的。当 context 被手动取消或者超时取消后，内部 chennel 关闭，通知所有使用当前 context 及其 child context 的 goroutine 进入生命周期结束的处理逻辑（一般用法是如此）。

context 提供两个函数 `Background()`, `TODO()` 来创建一个 emptyContext，主进程可以通过这两个函数来创建 root context。顺带一提这里的 emptyContext 其实就是通过 `type emptyContext int` 随后给该类型实现了 context 接口（但是函数里没有做任何操作）。

目前我用得比较多的是 `WithCancel()` 和 `WithTimeout()`，下面具体介绍一下他们的实现。

### WithCancel

`WithCancel` 中创建的 `cancelCtx` 才是第一个具备实际 cancel 功能的类型。后续的 `WithTimeout`， `WithDeadline` 都是复用它的 cancel 实现来做的。
```
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}

func newCancelCtx(parent Context) cancelCtx {
	return cancelCtx{Context: parent}
}

type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     chan struct{}         // created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}
```

代码片段中的`propagateCancel`函数会把当前创建的 context 加入 parent 的 children列表中；innermost 的 cancelCtx 还会启动一个 goroutine 检测 parent 是否结束运行，当发现 `parent.Done()` 之后，也将调用child的`cancel`函数中止 child context。

cancelCtx 不会自动进行 cancel，需要我们手动调用 cancel 来进行取消。一般来说可以用 `defer` 保证退出运行时执行 cancel 函数。


### WithDeadline

`WithTimeout` 封装了一下 `WithDeadline`，相当于 `WithDeadline(parent, time.Now().Add(timeout))`，所以这里我们还是来看 `WithDeadline` 的实现。

返回的是一个 `timerCtx` 内含一个 timer，超时之后自动调用 cancel。如果 child 的超时时间晚于 parent，则设为与 parent 一致。

```
type timerCtx struct {
	cancelCtx
	timer *time.Timer // Under cancelCtx.mu.

	deadline time.Time
}

func (c *timerCtx) cancel(removeFromParent bool, err error) {
	c.cancelCtx.cancel(false, err)
	if removeFromParent {
		// Remove this timerCtx from its parent cancelCtx's children.
		removeChild(c.cancelCtx.Context, c)
	}
	c.mu.Lock()
	if c.timer != nil {
		c.timer.Stop()
		c.timer = nil
	}
	c.mu.Unlock()
}

```
可以看到 `timerCtx` 的 cancel 也是复用 cancelCtx 的，除了 timer 以外没有增添其他东西。

WithValue不懂，懒得看了。

The End

--------