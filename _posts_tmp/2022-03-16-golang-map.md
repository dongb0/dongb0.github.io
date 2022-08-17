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

每个桶放8个元素，溢出则开辟 overflow 桶；overflow 桶数量过多则数组扩容


每个桶包含一个 tophash 字段保存hash的高8字节（还是高8位？

map 拉链法？使用什么hash函数？
