---
layout: post
title: "[Linux] mmap"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
catalog: true
tags:
  - Linux
---

`mmap`是linux提供的系统调用，用于将文件/设备映射到进程的虚拟内存地址中。随后进程可以像访问自己的内存一样来访问文件。

这是以前很多中文博客中对mmap的解释，多半是从英文文档中按字面意思翻译过来的，没有解释清楚跟`mmap`的实际作用。

比如为什么要“像访问自己的内存一样”访问文件？这有什么好处？什么场景适合用`mmap`、什么场景会带来overhead？这些问题我们一点点来看。

```
#include <sys/mman.h>

       void *mmap(size_t length;
                  void addr[length], size_t length, int prot, int flags,
                  int fd, off_t offset);
       int munmap(size_t length;
                  void addr[length], size_t length);
```

- Direct Access: It maps a file (or a portion of it) into a program's memory, bypassing explicit read and write system calls.
- Lazy Loading: It uses demand paging, meaning data is only loaded from disk into physical RAM when a specific memory location is actually accessed.
- Shared Memory: By using the MAP_SHARED flag, multiple processes can map the same file and communicate through it, creating an efficient form of Inter-Process Communication (IPC). 

根据上述介绍，使用`mmap`之后，读取文件数据不再需要通过读写系统调用。
> TODO：这能节省什么？切换到内核态的开销吗？具体是指什么样的系统调用？page fault有切换到内核态的开销吗？

`mmap`会按需加载数据，当数据不在内存时需要触发缺页中断，等待数据加载到page cache中。区别于普通read操作的是`mmap`不需要再从page cache拷贝数据到用户空间，而是由内核修改进程的页表，将虚拟地址指向刚才的page cache物理地址，减少了一次内核态到用户态的内存拷贝操作。

但既然是进page cache的，内存何时淘汰就由内核说了算，出现性能抖动时排查和调试都很复杂。这也是为什么数据库宁可自己用DirectIO+自己实现缓存管理，而不使用`mmap`。

## mmap的映射方式

`mmap`默认是映射一个文件，此外还可以通过`MAP_ANONYMOUS`直接将页表映射到物理内存，这也是一种常见的内存申请方式。使用`mmap`的场景有：

- MAP_ANONYMOUS + MAP_PRIVATE: 申请大块内存
- MAP_ANONYMOUS + MAP_SHARED: 进程间通信，比如父子进程共享内存
- fd + MAP_PRIVATE: 加载动态库
- fd + MAP_SHARED: 内存映射IO

> - MAP_SHARED: 修改对其他进程可见；
> - MAP_PRIVATE：修改对其他进程不可见；
> 前面我们说到，`mmap`是在访问时将进程的页表，指向文件所在的物理内存。那么我们可以让多个进程的页表都指向同一份内存，从而用fd+MAP_SHARED就可以实现page cache内“一份文件被多个进程共享”的效果。


除此以外还有：

巨页映射 (MAP_HUGETLB)：利用内核的 Huge Pages（大页）机制分配内存（如 2MB 或 1GB 的页），可以减少页表条目  
预热映射 (MAP_POPULATE)：在映射建立时就提前填充页表（对于文件映射会触发预读），从而避免后续访问时产生的缺页中断  
锁定映射 (MAP_LOCKED)：将映射的内存锁定在物理内存中，防止其被交换（Swap）到磁盘  


示例

```
#include <iostream>
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>
#include <cstring>
#include <sys/wait.h>

void anonymous_shared_example() {
    std::cout << "--- 场景 1: 匿名共享映射 (父子进程通信) ---" << std::endl;

    size_t size = 4096;
    // MAP_ANONYMOUS: 不关联文件
    // MAP_SHARED: 跨进程可见
    void* ptr = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED | MAP_ANONYMOUS, -1, 0);

    if (ptr == MAP_FAILED) {
        perror("mmap");
        return;
    }

    int* shared_data = static_cast<int*>(ptr);
    *shared_data = 100; // 初始值

    if (fork() == 0) { // 子进程
        std::cout << "[子进程] 读取初始值: " << *shared_data << std::endl;
        *shared_data = 200; // 修改共享内存
        std::cout << "[子进程] 已修改值为 200" << std::endl;
        munmap(ptr, size);
        exit(0);
    } else { // 父进程
        wait(NULL); // 等待子进程结束
        std::cout << "[父进程] 子进程修改后的值: " << *shared_data << std::endl;
        munmap(ptr, size);
    }
    std::cout << std::endl;
}

void file_shared_example() {
    std::cout << "--- 场景 2: 文件共享映射 (直接操作磁盘文件) ---" << std::endl;

    const char* filepath = "test_mmap.txt";
    int fd = open(filepath, O_RDWR | O_CREAT | O_TRUNC, 0666);
    
    // 必须先扩展文件大小，否则映射后写入会触发 SIGBUS 错误
    const char* text = "Hello Mmap!";
    write(fd, text, strlen(text));

    // 将文件映射到内存
    size_t size = 1024;
    void* ptr = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    close(fd); // 映射建立后即可关闭 fd

    if (ptr == MAP_FAILED) {
        perror("mmap");
        return;
    }

    char* file_content = static_cast<char*>(ptr);
    std::cout << "[读取文件内容]: " << file_content << std::endl;

    // 直接修改内存，相当于修改了 Page Cache，系统会自动刷回磁盘
    strcpy(file_content, "Mmap is fast!");
    
    // 显式同步回磁盘（可选，内核通常会异步处理）
    msync(ptr, size, MS_SYNC);

    std::cout << "[修改完成] 请查看 " << filepath << " 的内容" << std::endl;
    
    munmap(ptr, size);
}

int main() {
    // 场景 1：申请内存，用于进程间共享数据
    anonymous_shared_example();

    // 场景 2：映射文件，用于高性能文件读写
    file_shared_example();

    return 0;
}
```

## munmap, madvise

二者都可以用于释放内存，区别在于`mmap`会解除内存映射，释放对应的虚拟内存，如果进程再访问对应虚拟内存地址会直接segment fault；`madvise`释放物理内存但虚拟地址依然是完整的，后续如果再访问该地址，内核会重新触发缺页中断来分配新的内存页。

`madivse`有若干flag可以执行不同的操作：

- MADV_WILLNEED：告诉内核“我马上要用这段内存了”。内核会在后台异步地将数据从磁盘预读进 Page Cache，从而隐藏后续访问时的缺页中断延迟。
- MADV_SEQUENTIAL：建议内核进程将顺序访问这段内存。内核会激进地进行预读（read-ahead），并在读取后尽快释放前面已经访问过的物理页，避免污染 Page Cache。
- MADV_RANDOM：告诉内核访问是完全随机的。内核将关闭预读机制，避免浪费大量的磁盘 I/O 吞吐量。

--- 
The End