---
layout: post
title: "[]"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - cpp
  - programming
---

### 引

写这篇博文的动机来自于室友之前面试的一个提问：malloc和new最本质的区别是什么？室友回来问我们，大家七嘴八舌给了一堆猜测，但是好像当时都被面试官否决了，而面试官最后也没给出他心里的标准答案；

所以我一直想知道，malloc和new到底还有什么区别是我之前没想到的。

直到最近看代码，发现OB自己实现的allocator用法我一时居然没看懂它的使用方式（大概用法如下）

```
Allocator worker;
void *ptr = worker.alloc(sizeof(Obj));
Obj *obj_ptr = new (ptr) Obj();
obj_ptr->foo();
...
```

后来查了一下发现这是重载了new运算符，重载后的 `new` 可以自己定义它的行为，这里就是在`alloc`函数中分配好内存之后，再调用new时其实只是调用了构造函数而不再分配内存；OB这样设计是为了让worker来统一进行内存管理，减少内存泄漏的风险（虽然好像还是可能出现内存泄漏的问题）。

通过这个问题我才了解到（也可能是想起来？），C++里new是一个关键字，但同时new也是一个operator（二者同名）。当初学习能够重载的运算符里有new和delete，这两个能重载的其实是在头文件\<new>里定义的opeartor。而我们在使用 `Obj *ptr = new Obj();` 关键字new时实际上做了两件事：1. 用operator new分配内存； 2. 调用Obj类的构造函数[^0]。

所以二者本质的区别可能是这个：malloc只分配内存，而C++中的new（如果指的是keyword new的话）不单分配内存，还会调用构造函数（如果是一个类的话；内置类型int/char/double之类的就没有构造函数了）。

从这个层面上看，这一点可能才是当时面试官想听到的回答吧；到这里我才发现我对new好像完全不了解，甚至好像都没写过重载new的代码，赶紧的补一下。

### 重载new运算符


[1]: https://stackoverflow.com/questions/10513425/what-are-operator-new-and-operator-delete
[2]: https://zhuanlan.zhihu.com/p/354046948

[^0]: [StackOverflow - What are '::opreator new' and '::operator delete'?][1]