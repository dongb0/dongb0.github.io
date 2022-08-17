---
layout: post
title: "[]"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - programming
  - golang
---
一些杂项笔记：golang不是面向对象的，虽然它提供类似一些面向对象的编程方法，但是go没有封装（包级别的封装还是有的？）、没有类型的继承，类型B无法直接继承类型A中的方法，哪怕它们都实现了同一个接口。golang中如果类型B想复用类型A的方法，往往需要将类型A作为成员引入B，并且可以通过匿名类型的方法来// TODO 引用。

反射
获取类型、获取值、获取方法、调用方法





类型断言



### unsafe.Pointer

unsafe.Pointer 可以用来表示指向任意类型的指针，有些类似c语言中的 void 指针。比如通过以下代码片段来获取任意类型实例的地址：

```
// TODO： to be verified // 并不能，传过来的interface得到地址不同，为什么 // 因为golang的参数都是传值
func getPointer(x interface{}) unsafe.Pointer {
    return unsafe.Pointer(&x)
}
```
但是 unsafe.Pointer 跟 void 指针不同的是它不能直接进行算术运算，如果想用 unsafe.Pointer 进行运算的话需要先将其转化为 uinptr（实际是一个整型数），通过 uintpr 运算之后再转回 unsafe.Pointer。


那么为什么不直接用 uintpr？
能直接取 uintpr 吗？

uintpr 是一个无符号整型数，长度跟平台相关。 //  unsafe.Pointer 与平台无关吗？

普通指针能进行算术运算吗？

所以想要通过 unsafe.Pointer 对内存进行操作写法有点别扭，不过这是为了屏蔽底层的。。。可以理解。


// TODO
goroutine的栈动态变化，（那么会有 stackoverflow 吗？ 如果递归调用自己会发生什么？其他语言的栈是固定分配？c++貌似是的