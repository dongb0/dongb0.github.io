---
layout: post
title: "DNS Record Types"
subtitle: 
author: "Dongbo"
header-style: text
tags:
  - network
  - note
---

> 虽然一搜到处都是, 但是没写下来就是不过脑啊。~~写下来也没有过脑啊[ac01]~~

注册个人域名之后需要设置域名解析。越来越觉得之前的专业课和实验课是不是没有上过，什么都是"啊，这个我以前见过"，以及老师肯定知道，当初的我可能知道，现在的我......赶紧抢救一下。

| 常见类型  |     |
| --- | --- | 
| A (Address)    |  将域名指向一个IP地址     |
| CNAME (Canonial Name) | 将域名指向另一个域名，由另一个域名提供IP地址（如Github Pages使用自定义域名，CDN等）| 
| MX (Mail Exchange) | 记录一个域的邮件服务器地址，通常与一个A记录绑定 |
| NS (Name Server) | 指定由哪个域名服务器来解析 | 
| AAAA | 将域名指向一个IPv6地址 | 

有一说一，哪怕是现在我也没搞清楚这些记录各自的使用场景，只是在购买阿里云域名之后需要设置A记录将域名指向VPS服务器而已。