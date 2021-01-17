---
layout: post
title: "Linux Command (2) -- wget and curl"
subtitle: 
author: "Dongbo"
header-style: text
catalog: true
tags:
  - linux
  - cmd
---

> 简要比对wget和curl命令的不同，主要记录两个命令的用法，内容同样来自于man手册，若理解有误还请多包涵

## 写在正式内容之前的概述

wget 和 curl 都可以用来下载某个url的内容，比如网页、文件等。我在使用的时候并没有区分过，都是用简单的复制文件的链接地址然后敲下 `cmd [url]` 这样的格式命令来下载。

手册上的描述里这俩命令也比较相似：

> `curl` is a tool to transfer data from or to a server, using one of the supported protocols (HTTP, HTTPS, FTP, FTPS, SCP, SFTP, TFTP, DICT, TELNET, LDAP or FILE).   
`wget` is a free utility for non-interactive download of files from the Web. It supports HTTP , HTTPS , and FTP protocols, as well as retrieval through HTTP proxies.

根据手册来看，与 wget 相比，curl 不但能下载，还能向服务器发送数据；另外 curl 能够通过在url中使用变量，比较方便的批量进行传输，比如将可选的部分放在花括号中，如`http://site.{one,two,three}.com` 等效于  
`http://site.one.com, http://site.two.com, http://site.three.com` 
三个 url；   
还可以用方括号表示连续的数字/字母序列，如   
`ftp://ftp.numericals.com/file[1-100].txt`，但暂不支持括号嵌套的写法。

vertion 7.15.1 之后可以指定步长，如 `[1-100:10]` 可以请求第1个、第11个、第21个……等url。另外手册提到 `curl` 在传输多个文件时会尽量复用连接，来减少建立连接的overhand；但如果多次调用 `curl` 每次传输单个文件，则无法复用（因为每次都创建了新的curl进程）。

# wget

> 摘录一些可能会用到的参数

#### -b / --background

在后台执行，若没有使用 `-o` 指定日志的输出目录，则默认写到 `wget-log`。

#### -e *command* / --execute *command*

> Execute command as if it were a part of .wgetrc.

根据[Wgetrc Commands](gnu.org/software/wget/manual/html_node/Wgetrc-Commands.html#Wgetrc-Commands)，
`-e` 后面可接譬如 `logfile = fileName`、`connect_timeout = n`、`cookies = on/off`，而这些命令分别等效于`-o fileName`、`--connect-timeout`、`--cookies`等（后两个参数在手册里没有找到，只有`--no-cookies`）。所以可以姑且认为 wgetrc 命令相当于参数命令的另一种写法。但是很多命令并没有在手册种找到对应的参数，猜测可能一方面可能是命令可以提供最全面的控制方式，而通过参数只能进行一些常见的设置。

#### -V / --version

#### -v / --verbose


# curl

> 摘录一些可能有用的参数

大部分可设置为`True/False`的选项都可以通过`--no-option`，即在原本的选项前加上`--no`前缀来关闭选项。

另外，curl命令默认会显示一个类似下面列出的header来表示传输进度，可以使用`-s`选项关闭，也可以用`-#`选项替换成一个由`###`组成的进度条。

    % Total     % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload  Total   Spent    Left  Speed

#### -f / --fail

当curl传输因错误而终止时，不输出错误信息。常用于脚本中使用curl命令时。

#### -O (大写) / --remote-name

将下载的文件保存为与服务器上文件同一名称。

#### -o (小写) / --output <file>

输出到以`<file>`命名的文件中。如果使用的url中包含了`{}`，`[]`等变量，那么这里可以用`#`后接数字的方法来将每个url输出到不同文件中。比如

    curl http://{one,two}.site.com -o "file_#1.txt"

会分别使用 one 和 two 替换文件名中 `#1` 的部分,得到 `file_one.txt` 和 `file_two.txt` 两个文件;如果url中包含多个花括号或方括号,可以再接着用 `#2`, `#3` 等。


---------------

目前暂时只记了这么多，以后如果有学习到更多用法还会更新的。

The End