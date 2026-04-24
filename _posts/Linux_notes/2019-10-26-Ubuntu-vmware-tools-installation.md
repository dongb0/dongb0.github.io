---
layout: post
title: "Vmware -- Vmware tools intallation"
subtitle: 
author: "Dongbo"
header-style: text
tags:
  - linux
  - vamware
---

[Installing VMware Tools in an Ubuntu Virtual Machine (1022525)](https://kb.vmware.com/s/article/1022525)

1. 选择安装Vmware Tools后
1. 打开VMware Tools CD
2. 将`VmwareTools.x.x.x-xxxx.tar.gz`拷贝到桌面(随便哪里都行，执行时候进入对应路径就行)，解压，得到`vmware-tools-distrib`文件夹
3. shell执行`sudo Desktop/vmware-tools-distrib/vmware-install.pl -d`。重启虚拟机，安装完成。


