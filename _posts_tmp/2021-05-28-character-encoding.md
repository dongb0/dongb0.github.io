---
layout: post
title: "[Unicode] 字符编码：锟斤拷与烫烫烫"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - programming
---

> 烫烫烫还是以前大一学C++用VC 6.0经常忘记初始化数组的时候见得比较多。当时知道是字符编码的问题，但是一直没有深入了解到底为什么出现的是这几个字。不知道今天整理的笔记能不能回答大一懵懂无知的我提出的问题。


## GBK


## ISO/IEC 8859(TODO)[^2]

> //TODO：与编码方式区别？

是ISO和IEC定义的一系列8位字符集标准，目前包含15个字符集。每个字符集最多使用0xA0~0xFF的96个字符。


##  Unicode编码[^1]

Unicode编码的出现是为了解决ASCII、ISO8859-1等传统编码方式不能支持所有语种、不能在世界各地通用的问题。Unicode至今仍在不断修订，目前最新版本为2020年3月公布的13.0.0，收录了超过13万个字符。 // TODO：引用

  // 一般Unicode占用两字节，有些占用4字节（一些中文？）？？？待核对

Unicode在实现时，出于节省空间和对不同系统平台使用的考虑，可能会有不同的实现形式，称之为Unicode Transformation Format, UTF。以前经常见到的 UTF-8 就是 Unicode 的其中一种实现形式。

一方面，对英文使用1字节编码，其他字符1~3字节，减少编码长度。

  // 另一方面，大端，小端

  //TODO：UTF-8规则：


## 延伸1：编程语言中字符串的编码

// TODO： Java/golang/python/c++

## 延伸：

Byte of Mark，BOM

  // 待求证


  windows记事本

  UTF-8: EF BB BF (111011111011101110111111)
  UTF-16 LE: FF FE
  UTF-16 BE: FE FF
  UTF-32 LE: FF FE 00 00
  UTF-32 BE: 00 00 FE FF
  无BOM：隐式推断或用户手动指定
















[^1]: [Unicode Wiki](https://zh.wikipedia.org/wiki/Unicode)
[^2]: [ISO/IEC 8859](https://zh.wikipedia.org/wiki/ISO/IEC_8859)