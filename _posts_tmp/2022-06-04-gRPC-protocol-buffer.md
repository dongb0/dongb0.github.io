---
layout: post
title: "gRPC and Protocol Buffer"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - programming
---

之前看一些 golang 的项目时就碰到大量 proto buffer 的使用了，但是当时完全没有意识到文件夹里那一大堆注释着 `generated automatically, do not modify` 的文件到底有什么意义；甚至后来还以为 gRPC 是 golang 里特有的某种框架。年轻了。

今天大概看一下文档，搞清楚 gRPC 和 Protocol Buffer 都是啥吧。



RPC: remote procedure call
gRPC: google's RPC framework，但是官方文档上给出的 gRPC 全称是 gRPC Remote Procedure Calls，（又一个 Gnu is Not Unix 是吧）https://grpc.io/docs/what-is-grpc/faq/
or general purpose RPC？

简单理解的话，大概就是声明一个函数接口，然后其他程序（无论本机还是其他机器）都可以通过网络调用该函数，执行函数体中定义的操作，入餐和返回值同样通过网络传输。


protocol buffer：一种独立于平台和语言的数据格式定义框架，用于序列化和反序列化数据。文档说可以类比JSON，但是它比JSON更轻量更快。

比方说我们在 `.proto` 文件中定义如下的数据格式：

```
message Person {
  optional string name = 1;
  optional int32 id = 2;
  optional string email = 3;
}
```

然后可以通过 proto compiler 将其编译生成对应语言的 class，比如 C++ 的 。。。
Java 中的 。。。
Golang 中的 。。。

且提供序列化为 byte 数组和反序列化的函数，以及每个 field 对应的 getter/setter； seamless support of changes including the addition of new fields and the deletion of existing fields without breaking existing services.


不适用的情况：Object比较大，比如达到 MB 大小时；//TODO：more
