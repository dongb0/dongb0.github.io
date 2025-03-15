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
2. 内核的page是怎么实现的？是否有可以参考的地方。 linux内核的page结构体60byte，我这边要用96，目前看不改动结构可以精简到80，还能不能进一步压缩？

struct page里，用unsigned long flags中的bit来表示页面的各种状态；
目前也打算在OB的临时文件write cache里这样做的；

- slab是什么？
内存分配