---
layout: post
title: "Linux Command (4) -- netstsat"
subtitle: 
author: "Dongbo"
header-style: text
hidden: false
tags:
  - linux
  - cmd
---

> 更多内容可在[man手册][1]中查看，或者命令行输入 `man netstat`

简而言之，netstat是一个用来查看网络连接情况的命令/工具，这里摘录一些常用的选项。

- -r, --route 
    
    输出路由表

- -i, --interfaces=*iface*

    输出所有网卡接口，或者输出*iface*指定的接口

- -n， --numeric

    以 ip 地址的形式输出 socket 的 Local Addr/Foreign Addr 字段（否则会有域名和host name的形式）

- -a, --all 

    显示所有 listening 和 non-listening 的 socket

- -p, --program

    显示 socket 对应进程的 PID 和程序名称
    
通常我们想查看某个端口是否被占用时，可以用 `netstat -nap | grep <port>` 检查；另外我最近刚发现一个命令 `ss` 也能查看 socket，但是好像不显示 ip，有空再了解一下。

[1]: https://linux.die.net/man/8/netstat