---
layout: post
title: "[os] 文件系统"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
catalog: true
tags:
  - os
---

之前一直好奇格式化磁盘时选的 FAT32 和 NTFS 选项有何区别，当时知道 Linux 常用 Ext3/Ext4 文件系统，一般装双系统时如果 Linux 用的不是 NTFS 的话，在 Windows 没法访问 Linux 这边的文件，但背后原理一直没太搞明白，所以抽了点时间补一下文件系统的知识。

### 文件系统概念

// TODO：操作系统概念 - 文件系统

> 首先明确一下本节我们介绍的文件系统，在书上的标题为[^1]：基于inode的文件系统。inode 实际是类 Unix 文件系统中用于组织文件的数据结构，其中有代表性的是 Ext 系列的文件系统，如 Ext4。Windows 中使用的 FAT32 和 NTFS 以及其他文件系统我们将在文章的最后简要介绍。

我们知道虽然传统磁盘中存储数据是以 512B 的扇区为单位，但文件系统为了读写的效率，往往将多个扇区组成一个存储块（Block）来进行读写，一个存储块才是文件系统读写的最小单位，不满一个存储块大小的文件也需要占用一个块的大小。如何管理这些存储块，如何确定一个文件都存放在哪些存储块上，是文件系统需要负责的；在类 Unix 系统中，我们通过 inode 结构来完成上述任务。

> SSD中虽然没有物理扇区，但是为了兼容而引入了逻辑扇区

inode(index node) 是类 Unix 文件系统中用来描述文件系统对象信息的一种数据结构[^1],[^2]，具体而言它记录了文件模式、uid、gid、文件大小、访问/修改时间等元数据，以及最重要的，inode 记录了该文件所有存储块的位置信息；通过 inode 我们就能找到这个文件所有的存储块，这也正是其名称中 index 的作用：作为文件存储块的索引，帮助我们获取整个文件。

inode 采用类似分级页表的形式，来存储块的指针。我在\[[3]]中了解到 inode 大概是使用类似4级页表那样的4级指针结构。理解了 inode 的结构，有助于理解接下来要介绍的存储布局和文件描述符等概念。

// TODO：inode 示意图

> 思考：如果在 inode 的 meta 数据中记录各个级别的指针数目和位置，比如对于小文件我们记录使用了3个1级指针，没有2、3、4级指针；对于大文件，我们记录使用了5个3级指针和1个4级指针。这样inode 的空间利用是否会更灵活？比目前的做法是否会有别的损失？


#### 文件系统存储布局

// TODO：文件系统存储布局的示意图

文件系统在格式化时，会划分出超级块、块分配信息、inode分配信息、inode表和文件数据块等区域。**超级块**存放整个文件系统的元信息，如文件系统类型、版本、管理空间大小以及最大inode数量、空闲inode数量、最大块数量等统计信息。超级块之后存放的是**块分配信息**，使用 bitmap 记录文件数据块的使用情况；以及 **inode分配信息**，记录 inode 表的使用情况。随后是 **inode 表**，以数组的形式存放整个文件系统的所有 inode 结构，文件系统能够存放的文件数量也受此限制，可以使用 `df -i` 查看；最后的**文件数据块**自然就是实际存放文件的区域了。

```
$ df -i
Filesystem       Inodes  IUsed    IFree IUse% Mounted on
udev            2006969    641  2006328    1% /dev
tmpfs           2015263   1169  2014094    1% /run
/dev/nvme0n1p6  3489792 662443  2827349   19% /
```
关于 Ext4 文件系统支持的最大文件数量，\[[4]]中提及因为 Ext4 的 inode 编号是32位，因此理论上最大可支持 $2^{32} \approx 4×10^9$ 个文件，不过创建文件系统时需要设置 inode 表的大小，因此最大文件数量还是受限于设定的 inode 表的条目数量。

有了 inode 表，理论上我们就可以直接通过 inode 编号直接找到文件了。但是 indoe 编号与 IP 地址同样不便于记忆，于是恰如人们设计了域名来绑定某个具体 IP 的情况类似，操作系统提供了文件名到 inode 的映射，使得用户可以通过文件名来访问找到对应的 inode，然后访问文件。不过，文件名并不是存放在 inode 中的，而是存放在目录中。

目录也是一类特殊的文件，目录中保存的目录项记录着文件到 inode 映射的。而正因目录也作为文件存储，因此我们常常可以看到系统中的目录会占据 4K 的空间，这正是一个存储块的大小。

```
$ ll -a

drwxr-xr-x 43 batina batina 4.0K 4月  26 19:59 .
drwxr-xr-x  3 root   root   4.0K 8月  18  2020 ..
drwxrwxr-x  2 batina batina 4.0K 12月 21 10:45 .android
-rw-------  1 batina batina  12K 4月  22 12:40 .bash_history
```

另外，通过 `stat` 命令也能够查看文件/目录的 inode 信息。其中的`Size`字段是字节为单位的文件长度， `Blocks`是为该文件分配512B大小的块的个数，`IO Block`是文件系统存储块的大小。这里能看到，文件系统为 290B 的文件也分配了 8×512B=4K 大小的空间。

```
$ stat 404.html 
  File: 404.html
  Size: 290             Blocks: 8          IO Block: 4096   regular file
Device: 10305h/66309d   Inode: 530217      Links: 1
Access: (0664/-rw-rw-r--)  Uid: ( 1000/  batina)   Gid: ( 1000/  batina)
Access: 2022-05-08 20:09:10.852430180 +0800
Modify: 2021-11-07 09:51:10.269576059 +0800
Change: 2021-11-07 09:51:10.269576059 +0800
 Birth: -
```

#### 虚拟文件系统

上一节中我们了解了类Unix文件系统，但是我们还会有这样的疑问：平常使用PC时，我们似乎能够在同一台计算机上使用不同的文件系统。比如文章开头提到的，Ubuntu 系统中用的是 Ext4，但是却能够访问 Windows 的 NTFS 上的文件；另外早些年有些U盘还是 FAT32 的，也能在 Linux 中读写其中的数据。操作系统是如何支持同时使用不同的文件系统的呢？

这就要通过虚拟文件系统(Virtual File System, VFS)来完成了[^1]。我们知道不同的文件系统组织文件的数据结构和存储文件的方式不尽相同（现在我们还不知道，但是后面介绍了其他文件系统的存储方式之后就知道了），虚拟文件系统通过在内存中定义统一的数据结构，将不同文件系统的元数据转为VFS的数据结构，向上为应用程序提供统一的文件系统服务。

Linux 中 VFS 的数据结构同样包括超级块、inode、目录项等。VFS为每个文件系统在内存中维护一个超级块，并通过这些超级块对不同的文件系统进行管理；VFS在内存中还缓存有 inode 表和目录项，在适当的时候可以直接返回给应用程序；但书上有一块我不太理解的地方是提到VFS的 inode 是通过 radix tree 组织一个文件的数据，这个radix tree  构成了 VFS 的页缓存。检索到一个[相关的链接][2]，先码在这里以后再看。

#### 文件描述符

如果文件描述符单独拎出来讲的话，可能不易于理解和记忆，但是放在 inode 的介绍之后，介绍文件描述符就顺理成章起来了。

我们早就知道文件描述符实际上是一个整数，但是这个整数的实际含义是什么呢？实际上，每个进程内部一个文件描述符号表，表中每一项都对应一个文件描述结构，记录着一个被打开文件的inode、文件读写位置、打开模式等信息。文件描述符就是这个表的索引。

我们通过虚拟文件系统访问文件，首先需要在目录中使用文件名来解析文件路径，获取对应的 inode。为了避免进程每次访问文件都进行一次文件路径解析，而采用了文件描述符这一抽象，将解析得到的 inode 等信息存放在文件描述表中，使用文件描述符来访问，能一定程度提高文件读写的效率。

> 这其实是哈希表的一个应用。另外每个进程都默认创建文件描述符0为标准输入流，1为标准输出流，2为标准错误流。

// TODO：标准××流更多介绍

#### 页缓存和mmap

关于页缓存，目前了解到的是 Linux 内核能够将部分数据缓存在内核的页缓存区域中，应用程序读取数据时先通过 VFS 在页缓存中查找有无缓存，有则之间返回缓存的数据，无则从硬盘读入数据到缓存中，再返回给应用程序；写数据时，修改的数据并不一定会立即写入磁盘，而是存在页缓存上，并标为 dirty，由操作系统决定何时写入磁盘，比如页缓存空间已满需要被替换、操作系统每隔一段时间批量写入，或者可以通过调用 `fsync()` 要求操作系统立刻写。

使用页缓存，很大程度减少了最为耗时的硬盘I/O次数，虽然增加了内存中的数据拷贝这一额外负担，但是内存的读写速度比硬盘高两个数量级以上，所以文件操作中的瓶颈主要还是I/O。不过从上面描述的过程我们可以知道，使用页缓存意味着每次读写我们都会将数据从用户空间拷贝到内核的页缓存区域（读数据时则是从页缓存拷贝到用户空间），对某些自行实现了缓存管理的应用（如数据库）来说，操作系统的页缓存完全是额外开销。

// TODO：页缓存的配图  来自 https://juejin.cn/post/6956031662916534279

#### mmap

为了减少数据拷贝的过程，我们可以使用 mmap 直接将用户空间的某块虚拟内存映射到页缓存上。当我们需要访问数据时，由 VFS 负责将对应数据从磁盘中读入页缓存，应用程序通过共享内存的方式，可以直接访问和修改这些数据。

但是因为同样使用了页缓存机制，与缓存I/O类似，mmap 也不能保证数据能立刻写入硬盘中，如果要确保数据持久化，需要调用 `msync()` 来完成数据写入。使用 mmap 的一个缺点是，页缓存中数据页的写入和换出完全交由操作系统判断，而操作系统其实对应用程序数据的局部性是没有了解的，可能出现读写数据时产生大量的缺页异常，影响程序性能。所以数据库一般都会实现自己的缓存管理组件，而不是使用 mmap。

// TODO： mmap 的配图

> 当然用 mmap 的数据库也是存在的，比如 BoltDB，但是它只是一个几千行代码的简单kv存储引擎，性能有提升空间。

#### 直接I/O

除了 mmap，我们还可以通过打开文件时指定 O_DIRECT 标志使用直接I/O的读写方式。这样的I/O操作不会经过内核，应用程序可以直接在用户空间内与硬盘进行数据读取/写入交互，免去一次内存中数据拷贝的开销[^6]。

// TODO：memcpy 测试 4M 数据的平均拷贝速度
// TODO：以及配图

#### EXT4

Ext4 文件系统就是基于 inode 的文件系统，其基本原理（如 inode、VFS、页缓存等机制）在上面的章节中已进行了介绍。这一节中我们补充一些之前没提到的功能~~，可能还会介绍 Ext4 的一些新特性~~。

在\[[6]] 中我们了解到，尽管 Ext4 是 Ext 文件系统中最新的一版，但甚至它的开发者也认为 Ext 的设计比较老旧，只是作为过渡到下一代文件系统前的一个“临时版本”（缝缝补补又三年的意思吧我猜）。

Ext4 和 NTFS 文件系统都会通过日志来保证状态的一致性。没错，文件系统的写入也可以有日志的。~~我刚刚才知道~~
 
Ext4 有三种数据模式[^12]:
- journal mode  
  该模式下所有写入数据和元数据都先写入日志，然后再写到实际存储位置，这样一旦发生了崩溃，可以从日志中恢复数据，保证数据依然是一个一致的状态（不会出现 Partial Write 的情况）。这是 ext4 所提供最安全的数据模式，不过性能也是最差的，同时该模式下还无法使用延迟分配（delayed allocation），也无法使用 O_DIRECT 选项进行直接I/O。
- ordered mode  
  该模式下 ext4 只在日志中记录元数据信息，数据则直接写到对应 block，当日志提交之后才把元数据写到对应的 block；当发生崩溃之后，能够通过日志中的元数据回滚该页面。这样系统崩溃最多只导致正在写入的文件没能完整写入，而文件系统本身还是一致的；这是大多数 Linux 默认采用的数据模式[^13]。
- writeback mode 对写入操作不做任何额外保护，性能最好但也是最不安全的数据模式。

> journal 中存放的数据通常包括 inode, inode使用情况，目录数据块等[^14]。\[[7]]中有 ext4 日志格式的具体展示，但是太详细甚至到了琐碎的程度，没看懂。

在我理解看来，默认使用 ordered mode 意味着出现故障时，ext4 能够从日志中恢复出文件的元数据，也就是 inode 信息和目录信息，这意味着我们还能找到原本的文件（比如还能通过文件名找到对应的 inode，恢复的 inode 中也还记录着文件存放数据的 block 位置），但是不能保证文件中的数据是一致的（也许我们打开文件时发现文件损坏了），这种状态对用户来说是不理想的，但是对文件系统来说，起码它还处于一个能够正常工作的状态。如果元信息发生了损坏，那出现的后果可不仅仅是找不到文件：如果根目录损坏了，我们甚至无法在文件系统上进行任何读写操作。（个人观点）

// TODO: 如何查看当前 ext 文件系统所用模式
// TODO: delayed allocation

### 其他文件系统

#### FAT

// TODO：链式可能导致文件读取需要大量磁盘寻道，哪怕是SSD随机读写也比较慢

#### NTFS

// TODO：是否还是链式


#### ZFS

在一个 ZFS 的手册上看到，编者信誓旦旦的写道：*ZFS is designed to be the last filesystem you will ever need.*[^15]，ZFS 也是上面 ext4 作者提及的“下一代文件系统”，而我这里比较关心它如何保证文件读写的原子性，以及是否有其他新奇的特性。

通过前面的了解，我们知道其实 ext4 文件系统有能力保证文件写入的原子性（使用 journal mode），但是性能会下降得比较明显。而 ZFS 则直接通过 copy-on-write 机制来实现原子性。

##### RAID 入门

去看 ZFS 资料时候发现它们普遍都提到 RAID-Z，并解释说这是类似 RAID 5 的一种机制。可惜对我来说只是用一个不了解的术语来解释另一个不了解的术语罢了。于是决定看看到底什么是 RAID，本节也只是用于快速了解 RAID 概念，从各种博客和文档里东拼西凑来的。

RAID（Redundant Array of Independent Disks），译为独立磁盘冗余阵列，简要而言是将多块磁盘组成一个逻辑单元来进行文件读写，这样的好处是能够通过数据的冗余存储提供容错，亦或是在传输数据时能利用更多的通道，提高读写速度。RAID 根据层级的不同，可分为 RAID 0、RAID 1、... 一直到 RAID 6，另外还有混合的 RAID 01、RAID 10、RAID 50 等。目前了解来看 RAID 通常用在服务器集群中，可能是运维更需要掌握的技术。

###### 硬盘数据接口

介绍 RAID 之前，先插播一个小知识点：硬盘的数据接口。之前买电脑多少看到 ATA、SATA 等术语，只知道这些是某种数据接口的名称，却不知有何差别，也不知哪个更好，正好趁现在补一下。

- ATA[^16]，[^17]  
  ATA 有很多其他代称，如 IDE，PATA等，只是不同年代对同一技术的不同称呼而已[^19]，是较早使用的一种数据接口（1986年出现），全称为 Advanced Technology Attachment，有 ATA-1、ATA-2、ATA-3、Ultra-ATA/33、Ultra-ATA/66、Ultra-ATA/100、Ultra-ATA/133 等多个版本，目前已基本被 SATA 接口取代。从 ATA-3 开始支持 DMA 模式，几个版本的 Ultra-ATA 后面的数字表示最大传输速度，如 Ultra-ATA/33 使用 40 针引脚传输 2 Byte 数据（剩下位数大概是用于CRC校验），使用 60ns 的时钟周期，所以理论传输速度上限为 $1/60ns × 2 Bytes = 0.33×10^{10} Byte/s = 33 MB/s$[^18]。最后的 Ultra-ATA/133 是 2005 年发布的，只能支持 133 MB/s 的传输速度。

  > 总线带宽 = 总线频率 × 位宽

- SATA  
  SATA（Serial ATA） 在 2000 年发布，其修订版 SATA 1.0a 于2003年发布，使用 8b/10b 编码，传输速度达 150 MB/s[^20]。随后出现了 SATA rev 2(300MB/s)、SATA rev 3(600MB/s)，逐渐成为目前硬盘的主流数据接口。标准 SATA 使用7针引脚，其中发送/接收数据各使用2针；SATA 与 PATA 的最大区别在于 SATA 的数据是串行传输的（这也是其命名中 Serial 的来源），为什么串行传输的速度反而会比并行传输要快呢？

  综合几个网站上的介绍[^21],[^22],[^23]，大概得出以下结论：首先在相同频率下，并行传输的速度肯定是高于串行的；但是我们无法一直通过提高频率来提高并行传输的带宽，需要考虑以下两个问题：1）并行传输需要保证总线传输的数据同时到达（如果不是同时到达那读取时总会有部分数据缺失），而频率增加到一定程度之后我们很难实现这一保证；2）并行传输需要考虑串扰的问题，使用更高的频率也就更容易出现串扰，这往往需要更多时间来进行校验（最后几个版本的ATA使用更多的接地引脚也是为了减少串扰）。基于以上两个问题，并行传输的频率是存在上限的，而串行传输则不会出现这样的问题，因此串行传输可以使用更高的频率读写数据，也就能达到更高的读写速度。

  // TODO：8b/10b encoding

其他还有诸如 SCSI、SAS、FC 等数据接口，就留到以后再说吧，目前我了解到这里就够了。目前固态硬盘数据接口最常用的还是 SATA 3.0，但是随着传输速度的提升 SATA3.0 也慢慢难以满足需求了，于是出现了 PCI-E、M.2 等新的数据接口，此处不再展开，下次另开一坑专门介绍。

[^16]: [Wikipedia - 硬盘](https://zh.wikipedia.org/wiki/%E7%A1%AC%E7%9B%98)
[^17]: [Wikipedia - ATA](https://zh.wikipedia.org/wiki/%E9%AB%98%E6%8A%80%E8%A1%93%E9%85%8D%E7%BD%AE)
[^18]: [IDE（ATA） Bus Description](http://www.interfacebus.com/Design_Connector_IDE.html)
[^19]: [Difference bwtween ATA, PATA, IDE](https://superuser.com/questions/341452/whats-the-difference-between-ata-pata-and-ide)
[^20]: [Wikipedia - SATA](https://en.wikipedia.org/wiki/Serial_ATA)
[^21]: [Why is Serial Data Transimission Faster Than Parallel Data Transmission](https://www.howtogeek.com/171947/why-is-serial-data-transmission-faster-than-parallel-data-transmission/)
[^22]: [Why is SATA faster than PATA](https://forums.anandtech.com/threads/why-is-sata-faster-than-pata.1470879/)
[^23]: [Why is the SATA bus interface faster than PATA](https://www.quora.com/Why-is-the-SATA-bus-interface-faster-than-PATA-IDE)


了解这些其实最后只是想说多个硬盘可以并行传输数据（虽然可以一开始就这么说，但是现在知道了一些硬盘数据接口的发展历史难道不是一件很酷的事吗），因此组建 RAID 同时读写多块磁盘能够提升传输效率，所以让我们再回到 RAID 的话题上来，接下来看看不同层次的 RAID 有何异同。

> Data Striping[^24]: is the technique of segmenting logically sequential data, such as a file, so that consecutive segments are stored on different physical storage devices. 译为数据条带化，指将文件分为若干部分（Block），分别存放在不同的物理磁盘上，存取数据时可以同时读写多块硬盘，能够增加吞吐率；缺点是任意一块磁盘的损坏都会导致整个文件的损坏。

> 一点简单的概率计算：  
假设每块磁盘故障概率是0.1，则将文件存放于1块磁盘时，因磁盘故障导致文件损坏的概率是0.1；若使用5块磁盘进行条带化，则文件损坏概率是$ 1 - 0.9^5 = 0.4095$

- RAID 0[^25]  
  RAID 0 就是上面 Data Striping 的应用，但是因为没有数据的备份和校验，没有任何容错性。

- RAID 1  
  RAID 1 使用额外的硬盘用作镜像存储数据。以使用两块磁盘为例，存储数据时将数据同时写到这两块磁盘上，因此写入速度约等于将整个文件写入单个硬盘的速度，读取时可以同时读取两块磁盘上不同部分的数据来获得与 RAID 0 相当的吞吐率。比起 RAID 0 ，RAID 1 使用备份牺牲了一定的写入性能，换来了一些容错。

- RAID 2  
  暂时没有找到 RAID 2 比较详细的介绍，大多都说 RAID 2 很少用于实际生产中[^25]。其主要特点为加入汉明码作为校验，并且 Data Striping 的单位是按 bit 进行的（而不是 RAID 0 按 Block 划分）。汉明码本身只支持发现和纠正数据中1个比特位的错误，且 RAID 2 与 RAID 0 类似的是，任意一块磁盘故障都无法还原数据，所以这样的架构增加了存储的开销但是却没有增加太大的容错。

- RAID 3  
  同样来自 Wiki[^25]，RAID 3 按字节进行 Striping，并且使用一块磁盘专门存放校验码。这里的校验是指偶校验[^26]，因此可以采用异或运算来计算校验位。本身奇偶校验是无法进行纠错的，但是 RAID 3 中按字节条带化意味着任意一块磁盘损坏，丢失的数据都是已字节为单位的，能够用剩下磁盘上的数据再继续异或运算来恢复，来看个小例子[^26]：
  ```
  Disk 1: 1000 1011
  Disk 2: 0101 1010
  Disk 3: 1010 0001
  -----------------
  Parity: 0111 0000

  Suppose Data on Disk 3 is missing
  Recover by Disk 1, Disk 2 and Parity

  Disk 1: 1000 1011
  Disk 2: 0101 1010
  Parity: 0111 0000
  -----------------
          1010 0001
  ```

  我们知道奇偶校验在网络传输场景下是没法纠错的，因为发过来的数据包使用奇偶校验无法确定是哪些数据出错；但在 RAID 3 中我们只是假设磁盘故障导致数据丢失了而不是数据出错，而上面的例子也展示了 RAID 3 能容忍最多1块磁盘故障，依旧能保持数据完整。但这种按字节条带化的存储方式在并发读取数据时并不方便，因此更常用的是 RAID 5。

- RAID 4  
  RAID 4 是在 RAID 3 的基础上，将按字节条带化的存储方式改为了按块（Block）条带化，同时也使用一块磁盘单独用于存储校验数据。比起 RAID 3，RAID 4 能够同时响应更多的并发读取请求（RAID 3 因读取每块数据都需要访问所有硬盘来获取，所以并发读取在 RAID 3 中只能逐个完成）；但 RAID 3 和 RAID 4 都的所有读写都需要访问校验硬盘（验证或写入校验），因此该硬盘容易成为性能瓶颈，所以有了 RAID 5。

- RAID 5  
  了解了 RAID 3 和 4 之后理解 RAID 5 就方便多了，RAID 5 是将同样使用奇偶校验，但是不再使用单独的硬盘存储校验码，而是在文件数据写完之后同样按照条带化的方式接着存储校验数据。这样校验数据就不会单独存放在1块硬盘上，而是分布在整个阵列的硬盘中。虽然我没有实际的使用经验，但大多数资料都说 RAID 5 是最常用的。

- RAID 6  
  RAID 6 则是在 RAID 5 的基础上再加入一个校验，能容忍最多2块硬盘的数据丢失。// TODO
  
- RAID-Z in ZFS[^27],[^28]   
  根据 Wiki 描述， RAID-Z基本架构与 RAID 6 非常类似，同样是将校验数据分散存放在条带化数据之后，但 RAID-Z 可以动态改变条带化的 Block Size，以及增加了元数据的校验和。


[^24]: [Wikipedia - Data Striping](https://en.wikipedia.org/wiki/Data_striping)
[^25]: [Wikipedia - RAID](https://en.wikipedia.org/wiki/RAID)
[^26]: [Wikipedia - Parity](https://en.wikipedia.org/wiki/Parity_bit)
[^27]: [What is RAIDZ](http://www.raidz-calculator.com/what-is-raidz.aspx)
[^28]: [Wikipedia - Non-Standard RAID Level](https://en.wikipedia.org/wiki/Non-standard_RAID_levels)
[^29]: [Wikipedia - Copy on Write](https://en.wikipedia.org/wiki/Copy-on-write)

#### ZFS Feature

原本还想更深入了解一下 ZFS，但是被太多前置知识分散精力了，这里就简单罗列一下 ZFS 的**部分**特性吧，当作拓展知识面了。

- Copy-On-Write[^29]  
  这是我们已经很熟悉的一种技术了，在虚拟内存（如 Linux fork() 之后的共享内存）和存储方面都有应用（如之前介绍过的 BoltDB），现在 ZFS 也采用 COW 来保证数据的完整性。但关于 ZFS 与其他采用日志的文件系统之间的性能对比目前没有查阅到更多资料，因此这里只了解到 ZFS 通过 COW 保证数据的一致，并且这也让 ZFS 拥有提供文件快照的能力。

- Snapshot  
  如上所述，使用 COW 的 ZFS 很方便的能够提供文件的快照，有了快照我们可以很方便的进行恢复已删除文件、回滚文件到之前的版本等操作。但是代价当然是快照要占用额外的存储空间，之前我一直好奇这些快照占用的额外空间应该如何归还，后来在手册上看到可以通过给快照设置失效时间，过期之后再清除[^15]。

- RAID-Z & VDEV  
  在 NFTS 中我们会将硬盘划分为若干个卷（volume），分成 C盘、D盘等，这在 ZFS 中称为 pool。一个 pool 由若干 VDEV （virtual device）组成，一个VDEV由若干物理硬盘组成；因此可以根据每个 VDEV 的物理硬盘数量配置 RAID-Z 的级别：如果每个 VDEV 只使用1块硬盘，则与目前的文件系统几乎没有区别；如果每个 VDEV 使用2块硬盘做镜像存储，则相当于 RAID 1；还可以配置成 RAID-Z1 等。从手册上看，用户似乎可以根据需要和硬件条件来配置适合自己的的 RAID-Z 级别，把一项用于服务器集群中的技术塞进了 PC，让我们的 PC 变得更加强劲和安全了~~，这使你充满了决心~~。    
  // TODO: In particular, RAIDZ does not have a write hole.


**小结**： 从架构设计上能看出 ZFS 的野心：采用128位的架构支持ZB级别的数据存储、通过 VDEV 的方式提供类似 RAID 的容错和高吞吐、使用数据备份和校验提供更高的数据完全和完整性等，听上去确实像是下一代文件系统该有的样子。不过目前对 ZFS 实际使用的性能表现和可用性了解还不多，甚至对于很多新特性比起现在方案是否会引入的新问题如何解决还不清楚，比如使用 Copy-On-Write 后何时清理旧数据、使用 RAID-Z 磁盘利用率为多少，一般PC需要使用多少块硬盘比较合适、成本会增加多少。

提供快照意味着会消耗一定空间存储旧文件，使用 RAID-Z 也需要更多的硬盘，对于目前PC常见的1到2块512G 到 2T 的硬盘来说，这样的额外消耗不知道是否能被一般家庭接受。有了更快的 SSD 之后大家装机肯定更倾向于选择 SSD 而不是便宜的机械硬盘，但 ZFS 在剩余空间较少的情况下性能会明显下降，因此成本和性能之间的平衡也是需要考虑的。这么看来，ZFS 确实是面向“未来”的文件系统：现在还不是时候。

以上都只是我个人的瞎吉尔分析，我也不能确定到底什么因素影响 ZFS 的普及，使用的复杂性、硬件成本还是操作系统的支持问题？也许是它还没发展到一个比较成熟的程度，或者是人们对这样一个终极文件系统的需求并不迫切，我们还是就这样静静观望它的发展吧。

### 扩展：硬盘扇区从 512Byte 到 4K 的改变

传统磁盘的扇区大小是 512 B，但是现在 Advanced Format 正在引入 4K 大小的扇区来提供更高的磁密度和硬盘容量，以及更加强大的错误纠正功能[^4],[^30]。使用 Advanced Format 使得原本每个 512B 扇区使用 50B 纠错码减少为每个 4K 扇区使用 100B 纠错码，增加了磁盘可用空间。另外采用了新的纠错算法也提升了检测和纠正错误的能力（具体如何纠错没有深入了解）。

不过由于很多软硬件还是基于 512B 大小扇区实现的，为了兼容，硬盘制造商提供了模拟 512B 扇区的功能，在读取某个 512B 大小的扇区时，会将包含该扇区的整个 4K 数据全部读入，然后取出其中的 512B 提供给主机；但写入过程会稍微复杂，写入一个 512B 扇区同样需要读取 4K 数据，修改其中内容，然后再将数据写回磁盘，主要问题出现在如果磁盘分区不对齐，比如若干 512B 逻辑扇区没有对应 4K 物理扇区的开始地址，而是对应第2到8个逻辑地址，那么很容易出现反复读取并写入的情况，导致性能的下降。

// TODO：逻辑扇区和物理扇区不对齐的示意图

目前 Windows 或 Linux 均已能够识别 4K 的高级格式化标准，只要使用较新版本的操作系统则不用担心上述问题[^4]。

[^30]: [Wikipedia - Advanced Format](https://zh.wikipedia.org/zh-cn/%E5%85%88%E9%80%B2%E6%A0%BC%E5%BC%8F%E5%8C%96)

### 扩展：SSD的扇区

// TODO: 

### 附录：缓存I/O、直接I/O和 mmap 文件读写速度对比

看完前面的理论之后，果然还是想编码验证一下不同的 I/O 方式是否符合我们的分析；同时还能顺带了解一下一般PC的磁盘读写速度。

在 coding 之前，根据现有知识我们可以做出以下猜想：

- 硬盘读写速度比内存慢两个数量级以上（SSD）
- 直接 I/O 的读操作应该明显慢于缓存 I/O 和 mmap（直接I/O是从硬盘中读，另外两个是读内存的页缓存）
- 直接 I/O 和 mmap 写入应该比缓存 I/O 略微快一点（少一次内存中数据拷贝）

然后简单编写了几段文件读写的代码在本机进行了测试，先生成了几个4MB的文件，然后用各种 I/O 方式读写100次，多次运行统计后，取其中一次比较典型的运行结果，如下表所示：

> 源码可在[这里](https://github.com/dongb0/IO_Speed/tree/main)查看

|      | Write(400MB)/sec | Read(400MB)/sec |  WR(各400MB)/sec |
|------|:--------------:|:-------------:|:--------------:|
|缓存I/O| 1.625 | 0.076 | 1.652 |
|直接I/O| 1.572 | **0.034**|  1.714  |
|mmap  | 1.606 | 0.079| 1.624 |

直接 I/O 的写操作确实比缓存 I/O 和 mmap 的写入略微快一点，根据表中的数据我们可以验证猜测 1 和 3；但是不知为何在我电脑上测试得到直接I/O的读取速度比另外两个都快，与猜测 2 不符，有待进一步验证。

##### 踩过的坑

简单搞点文件读写也能踩坑我是没想到的，缓存I/O直接open/read/write，没碰到啥问题；但是直接I/O碰到写文件一直失败的问题，后来查到说直接I/O存放数据的缓存区要与磁盘逻辑块大小对齐[^7]，直接在栈上开一个数组或者用 malloc/new 在堆上开 buffer 都不能保证对齐，所以写入一直失败；需要使用 posix_memalign()/aligned_alloc()/memalign()/valloc()/pvalloc() 等函数分配内存[^8]。我一试，果然啊，马上就成功写入了。

另外其他几个坑大概是：

- mmap不会自动扩展映射文件的大小，因此如果想写入一个空文件，或者要写入的内容大于原本文件的大小，需要使用 fallocate() 或其他函数先给文件扩容；否则写入时会出现 Bus Error[^9]。
- mmap 映射文件的 flag 参数我最初参照一些 mmap example 设为了 `MAP_PRIVATE`，发现文件里没有写入数据，查文档看该参数的含义是：*Create a private copy-on-write mapping.Updates to the mapping are not visible to other processes mapping the same file*，也没发现问题；后来找了个完整的 example 运行之后对比才发现[^10]，flag 应该设置为 `MAP_SHARED`。这时候才发现，看[文档][5]时漏了一句:`MAP_PRIVATE` *... updates are not carried through to the underlying file.*。就是说 `MAP_PRIVATE` 不会把修改写入文件，可能只能用来读取。文档没看完就瞎写，属实憨批。
- 另外在写直接I/O时，我想既然用了动态内存分配，那用上智能指针保证不出现内存泄漏比较好吧，但是用了 unique_ptr 之后，原本写400M数据2秒不到，现在变成6秒了，是我实现上哪里出了问题吗...为了控制变量，最后还是没用智能指针，手动释放的内存。

------------

### Summary

发现除了银杏书[^1]以外，阮一峰11年就有博客介绍 inode 了[^11]，内容的组织也蛮类似~~（*我有一个大胆的猜想，没错我猜是这一章是阮一峰写的，你以为我想的是什么*）~~，开个玩笑，书上还是扩充了很多内容的。在这篇博客的写作过程中，除了学习书上介绍的 Ext 和 FAT/NTFS 文件系统，我额外看了 ZFS，并顺带了解几种数据接口的特点和发展历史，以及 RAID 的架构、优势；最后还编码测试了 inode 文件系统的缓存I/O、直接I/O和 mmap 3中不同的文件读写方式的速度，不过测试结果还有部分与理论分析不符，目前水平暂时也没发现导致这一差异的原因。

虽然在文件系统这方面，感觉自己还只是初学者水平，但比起之前的我，起码也有更深入一点的理解了吧。

The End

----------

### Ref

[^1]: [现代操作系统 原理与实现 - 第9章]()
[^2]: [Wikipeida - inode](https://zh.wikipedia.org/wiki/Inode)
[^3]: [Wikipedia - Extended file system](https://en.wikipedia.org/wiki/Extended_file_system)

[^4]: [过渡到高级格式化4K扇区硬盘](https://www.seagate.com/cn/zh/tech-insights/advanced-format-4k-sector-hard-drives-master-ti/)
[^5]: [美团 - 磁盘I/O那些事儿](https://tech.meituan.com/2017/05/19/about-desk-io.html)

[^6]: [博客园 - 磁盘I/O的三种方式对比：标准I/O、直接I/P、mmap](https://www.cnblogs.com/longchang/p/11162813.html)
[^7]: [博客园 - Direct I/O使用](https://www.cnblogs.com/muahao/p/7903230.html)
[^8]: [Linux man-pages: posix_memalign](https://man7.org/linux/man-pages/man3/posix_memalign.3.html)
[^9]: [StackOverflow - mmap bus error]((https://stackoverflow.com/questions/20587935/bus-error-opening-and-mmaping-a-file))
[^10]: [StackOverflow - mmap example](https://stackoverflow.com/questions/26259421/use-mmap-in-c-to-write-into-memory)
[^11]: [阮一峰的网络日志 - 理解inode](https://www.ruanyifeng.com/blog/2011/12/inode.html)
[^12]: [Ext4 File System Ducument](https://www.kernel.org/doc/Documentation/filesystems/ext4.txt)
[^13]: [Understanding Linux filesystems: ext4 and beyond](https://opensource.com/article/18/4/ext4-filesystem)
[^14]: [Ext4 Journal](http://ext4magic.sourceforge.net/journal_en.html)
[^15]: [Intro to ZFS](https://www.ixsystems.com/community/resources/introduction-to-zfs.111/version/164/download)


[2]: https://www.kernel.org/doc/html/latest/filesystems/vfs.html
[3]: https://linoxide.com/linux-inode/
[4]: https://serverfault.com/questions/104986/what-is-the-maximum-number-of-files-a-file-system-can-contain
[5]: https://man7.org/linux/man-pages/man2/mmap.2.html
[6]: https://opensource.com/article/18/4/ext4-filesystem
[7]: https://ext4.wiki.kernel.org/index.php/Ext4_Disk_Layout