---
layout: post
title: "[libp2p源码阅读] - 1 TcpTransport"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - libp2p
  - source code
---

TcpTransport是libp2p节点创建连接时的connection wrapper class，它首先调用标准库的net.Dialer创建一个普通的socket连接，然后通过Upgrader将连接升级为安全、多路复用的连接

安全连接是给普通连接加上（PSK\私钥加密

多路复用是通过yamux或mux？两个多路复用库，原理是？

