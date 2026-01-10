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

关于Linux内存页，想了解几个部分：
1. 内存管理有关回写page、write back线程的部分。
2. 内核的page是怎么实现的？是否有可以参考的地方。

> linux内核的page结构体60 byte；我写临时文件 write cache page 却达到96，想看下哪里可以借鉴。

参照[博客1](https://zhuanlan.zhihu.com/p/573338379) 和 [博客2](https://blog.csdn.net/u013711616/article/details/141338615)
我们可以知道这个60字节的page结构体是内核用来描述一个4KB物理内存页面的，但内核中有各种各样的场景，page也有各种状态。  
// TODO：例子

因此内核中使用unsigned long flags中的bit来表示页面的各种状态，在某些bit设置为true时，对应的字段才是valid的。

在运行过程中根据flags的状态来判断需要使用哪些字段，相应的也就会给代码维护增加一定的复杂度。

## Page Cache相关


- slab是什么？
内存分配