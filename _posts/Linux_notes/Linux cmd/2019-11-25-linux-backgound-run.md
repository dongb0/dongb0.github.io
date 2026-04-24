---
layout: post
title: "[Linux] 关于后台运行程序"
subtitle: 
author: "Dongbo"
header-style: text
tags:
  - linux
  - note
---

> 关于后台运行程序，胡乱查到的一些关于 service, tmux, screen, bg, & 等的说明

因为梯子需要在 vps 上挂着 caddy，要把 caddy 转入后台运行，所以这里了解了一下 linux 后台执行程序的一些方法。

### service 

根据目前的理解，将一个程序注册为 service 是要编写一个启动脚本，然后由系统启动时运行该脚本完成的。但是我怎么知道 caddy 启动需要什么配置环境、如何设置参数啊（挠头。如果我直接把 `caddy run` 命令写进 service 的脚本里能不能行呢...

Update：已经找到 caddy 注册 service 失败的原因了，原来是下载 caddy 的时候要选择额外 hook.servie 插件，我当时没选就按照教程的执行命令，难怪总提示找不到参数。


### tmux

因为最开始按照文档把 caddy 注册成 service 莫名其妙的失败了，决定暂且先用tmux顶上能用上再说。简单查了一下，tmux 相当于可以创建一个与当前终端分离的新会话，当前终端的退出或其他操作不会影响其他tmux会话。

又是[阮一峰的tmux介绍][1]

这里简要记几个可能比较常用的命令。 

- `tmux new -s <session-name>`  
  新建一个命名 session，在该会话中启动我们需要放在后台运行的任务
- `tmux detach`  
  分离当前会话，此时我们可以退出当前终端，而刚才启动的任务就会在被分离的会话中继续运行
- `tmux kill-session -t <session-num/session-name>`  
  结束某个会话，也可以在某会话中输入 `exit` 来退出
- `tmux rename-session -t <old_name/old_num> <new_name>`  
  重命名session
- `tmux ls`  
  列出当前所有会话

另外 tmux 还有很多快捷键简化输入上述命令，但是容易遗忘，此处赞不罗列，需要时可 `man tmux` 查看

### screen

screen 的功能与 tmux 类似，因为本文只是想了解有什么能用于后台运行进程，故不在此对照 tmux 和 screen 的优劣。

命令与 tmux 多有相似。

- `screen -S <session-name>`  
  创建新session
- `screen -d`  
  detach screen sessin
- `screen -r <session-name>`  
  resume session
- `screen -ls`  
  list session

没找到结束 screen 的命令只有快捷键，看来暂时只能进入对应session来退出。

### &

`&` 似乎来自于 bash 提供的 Job Control 功能。在某条命令结尾加上 `&` 符号，可以在后台运行该命令，实际上 shell 是启动一个 subshell 来完成，所以当前 shell 退出之后这个后台命令也会终止运行。不适用于我当前需要场景。

另外提一嘴，输入`^Z`（control+Z）挂起的命令可以通过 `jobs` 查看，然后使用 `fg %[Job_Number]` 恢复前台运行，`bg %[Job_Number]` 放到后台运行。

### bg

类似 `&`，但是只能用于被挂起的 jobs，将其转为后台运行；我们还可以使用 `fg %[Job_Number]`将一个后台运行的 job 放到前台执行。更多说明可输入命令 `man bash`，查看 SHELL BUILTIN COMMANDS secttion。

The End

---------------

[1]: http://www.ruanyifeng.com/blog/2019/10/tmux.html
