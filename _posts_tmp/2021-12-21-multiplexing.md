---
layout: post
title: "[network] 通过net.Conn和go-libp2p-yamux认识多路复用"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - network
  - golang
---

上次听到“多路复用”这个词还是大二时候的机网课，一晃好几年了啊。


### TCP/IP的多路复用


### go-yamux的多路复用

跟上述的多路复用并不完全一样，简要来讲go-yamux这个多路复用库是将一个连接拆成多个逻辑上的stream，每个stream的使用方法都与原来的net.Conn一致，但是这些stream底层其实共用一个连接来收发数据。所以说是逻辑上的流。

这样做是为了什么？
1.建立连接的overhead。假设节点A有两个线程需要跟节点B收发数据，如果没有多路复用，那么需要建立两个Connection；但是现在只需要建立一次Connection，然后通过它创建两个Stream来进行数据传输。上层应用达到了使用两个逻辑上Connection的目的，而我们底层其实只创建了一个连接，但是这样的细节没必要暴露给上层组件。

为什么libp2p还需要引入多路复用

猜想：因为libp2p不一定使用TCP/IP？
1. 应用本身的需要，
2. 为了NAT穿透