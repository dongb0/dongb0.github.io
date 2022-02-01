---
layout: post
title: "Linux tmux"
subtitle: 
author: "Dongbo"
header-style: text
tags:
  - limux
  - note
---

> 关于后台运行程序，注册service, tmux, screen, bg, &等

因为梯子需要在 vps 上挂着 caddy，要把 caddy 转入后台运行，所以这里了解了一下 linux 后台执行程序的一些方法。


### tmux

因为最开始按照文档把 caddy 注册成 service 莫名其妙的失败了，决定暂且先用tmux顶上能用上再说。

又是[阮一峰的tmux介绍](http://www.ruanyifeng.com/blog/2019/10/tmux.html)

这里简要记几个我可能比较常用的命令。


### service 

Update：已经找到 caddy 注册 service 失败的原因了，原来是下载 caddy 的时候要选择额外 hook.servie 插件，我当时没选就按照教程的执行命令，难怪总提示找不到参数。

### bg


### &
