---
layout: post
title: "[Linux] IO模型"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
catalog: true
tags:
  - Linux
---


> 本意是想学习Linux的异步IO模型，而发现IO模型还有很多内容要补，故做此文。PS: gemini从《Unix网络编程》概括得到大纲，后续检索补全扩充内容。

## 引言

本以为Linux的IO模型只有同步和异步两种，检索之后发现比我想象的要多；可能以前上学的时候匆匆扫过，没在脑子里留下什么印象。现在来补补课吧，IO模型有下列5种：

一次完整的读操作（比如 read 或 recvfrom）会经历以下两个阶段：

1. 阶段一：等待数据准备就绪 (Wait for data)
  - 例如：等待网络上的数据包到达网卡，并被复制到内核的缓冲区中。

2. 阶段二：将数据从内核空间拷贝到用户空间 (Copy data from kernel to user) 
  - 一旦内核缓冲区有数据了，系统需要把这些数据拷贝到应用程序的内存（用户空间）中，应用程序才能处理它们。

## 阻塞 I/O 模型 (Blocking IO)

最传统的IO模型，进程需要等待阶段一和阶段二执行完毕，才会被唤醒，接着往下执行其他操作。

read/write 就是采用这种工作方式。

// TODO：sendfile绕过用户态直接发送数据，属于哪个类型？

## 非阻塞 I/O 模型 (Non-blocking I/O)

进程在阶段一不断询问内核数据是否准备好（轮询 Polling），如果没有，则内核会返回一个EAGAIN错误避免阻塞；在阶段二阻塞等待拷贝数据。

```
#include <fcntl.h>
#include <unistd.h>

// 以只读且非阻塞的方式打开一个命名管道或设备文件
// 注意：这对普通磁盘文件往往没有实际的非阻塞效果
int fd = open("/dev/tty", O_RDONLY | O_NONBLOCK);
```

注释里说文件设置这个 flag 没有效果，是因 Unix 将设备分为慢速和快速两类，socket 算慢速，磁盘算作快速的，设计VFS时没有设计返回 EAGAIN 的机制。

如果是 Buffered IO，则read需要去 page cache 找，命中则直接返回；未命中，仍旧会让线程陷入睡眠，来等待数据从磁盘中加载上来（即使设置了非阻塞flag）。

如果是Direct IO，那内核会直接让线程陷入睡眠等待IO操作结束。

而 socket 则会在没有数据时立刻返回 EAGAIN：

```
#include <sys/socket.h>
#include <netinet/in.h>

// 直接创建一个非阻塞的 TCP 套接字 (注意 SOCK_NONBLOCK)
int sockfd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);
```

对这个 sockfd 调用读写操作，如果出现没数据或者缓冲区满，就会返回EAGAIN。

但要注意的是，此类IO模型虽然叫非阻塞，但并不能实现异步IO。目前搞不清楚这个命名是否是历史发展的遗留产物，有些容易造成误解。

## I/O 多路复用模型 (I/O Multiplexing)

高并发网络编程常用IO模型，select、poll、epoll等都属于这一类。

// TODO：例子，也需要展开说明

## 信号驱动 I/O 模型 (Signal-Driven I/O)

该模型下，进程不会阻塞在阶段一，而是直接返回，等到系统发送信号后再回来执行二阶段的数据拷贝。

// TODO：例子

## 异步 I/O 模型 (Asynchronous I/O - AIO)

> 本文学习的重头戏：Linux的异步IO模型，POSIX最早期（优缺点是什么），io_submit（限制是什么），最后的io_uring有什么优点

### POSIX

早期的Linux没有支持异步IO，而是通过glibc库，在用户态创建了一个后台线程池去做IO，提供aio_read/aio_write借口，底层实现还是阻塞IO。

缺点是性能差，高并发时线程上下文切换开销大。

### io_submit


Linux 在 2.6 版本引入了真正的内核级异步 I/O，通过libaio库调用，提供io_submit/io_getevents接口。内核会在后台完成硬件交互后，再通知用户进程。

缺点是只能支持O_DIRECT，如果是buffered IO还会退回到阻塞模式，并且只支持磁盘文件，不支持网络IO。

此外提交和获取结果也需要用户态、内核态切换。

> 这也是目前数据库用得较多的异步IO模型，完美契合数据库的需求。

// TODO：跟POSIX有什么区别？

### io_uring

2019年 Linux 核心开发者 Jens Axboe（块设备维护者）重写的一套全新的异步 I/O 框架 io_uring。

原理：它在用户态和内核态之间创建了两个共享内存的环形缓冲区（Ring Buffers）：
- SQ (Submission Queue)：用户态程序把 I/O 任务放到这里。
- CQ (Completion Queue)：内核把完成的结果放到这里。

优点：支持 Buffered I/O、Direct I/O、网络 Socket 读写（send/recv）、甚至是 accept 和 epoll。无需用户态和内核态之间的上下文切换，性能优于io_submit。

// TODO：为什么不需要上下文切换；用了mmap将用户态和内核态的队列都映射到了同一块内存。加上内核轮询。

> 也可以理解为什么OB没有采用，因为对OB来说是新东西，没有动力去做；况且对数据库来说收益也并不大。