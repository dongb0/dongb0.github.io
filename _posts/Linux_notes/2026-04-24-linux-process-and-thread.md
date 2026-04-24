---
layout: post
title: "[Linux] All you should know abot Process and Thread"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - Linux
---


## 进程和线程

// TODO：定义

### 进程间通信(IPC)


### // TODO：网络协议



> TODO: 什么是用户态和内核态；英文是什么；是否单独开一篇博客来记录；
用户态（User Mode）和内核态（Kernel Mode）是操作系统为保护系统安全而划分的两种 CPU 运行级别。内核态拥有最高权限，可访问所有硬件和内存；用户态权限受限，只能运行应用代码。它们通过系统调用或中断切换，以保障稳定，用户程序申请资源时会切换到内核态。
我可以简单理解为，用户态就是linux内核有一段内存空间，存放着内核代码的函数，并且拥有系统权限。进入内核态，就是从用户态的函数执行流程里，切换到内核函数执行，主要开销相当于函数调用（可能出现打断流水线、TLB失效、上下文切换等开销）

> 涉及到虚拟内存、进程等操作系统，尚未完全理解（没有太深刻的印象）。需要再补补课。

