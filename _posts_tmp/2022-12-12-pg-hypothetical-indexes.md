---
layout: post
title: "[]"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - database
---

1 PG hypo index 用法示例； // 官网就有，google可查
2 大概原理；

// 优化器需要估计使用（虚拟）索引后的开销是不是更小，这需要能够评估 hypo index 带来的增益；看看这是如何做的。

// 大概估算使用索引后进行的 IO 次数？//如何估算
