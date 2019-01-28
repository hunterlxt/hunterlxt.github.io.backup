---
layout:     post
title:      "操作系统的三个主题——持久化"
subtitle:   "The part 3 of Operating Systems: Three Easy Pieces"
date:       2019-01-27
author:     "TXXT"
catalog:    true
tags:
    - Operating System
---

> 参考：*Operating Systems: Three Easy Pieces*
>
> OS是一个工程化色彩很强的CS基础课程，因为跨专业的原因，没有完整学习过这门课程，现在还债了……

# 36 I/O Devices

## 36.1 System Architecture

开始讨论前，看一下典型系统的结构。一个CPU连接到了 **memory bus**。一些设备通过 **General I/O bus** 连到系统中，一般是 **PCI** （或者代替者）；高性能 I/O 设备就在这条线上。剩下的就是慢速设备用的总线，**Peripheral I/O Bus**就是干这个工作的。

![1547039817237](/img/in-post/持久性Persistence.assets/1547039817237.png)

那为什么要用这个层次结构？因为一个总线越快，那么越短，高性能的内存总线没有空间去插其它的设备了。此外，构建高性能的总线需要更多的成本。

当然，现代的系统慢慢使用特制的芯片组和更快的点对点互连提升性能（下图）。这是 Z270 芯片组的结构。eSATA接口用来连接磁盘。**PCIE**(Peripheral Component Interconnect Express)用来连接更高性能的设备，例如网卡、NVME的SSD等。

![1547527623499](/img/in-post/持久性Persistence.assets/1547527623499.png)

## 36.2 一个规范设备

这里说的不是真正的设备，而是便于理解。下图看出两个重要的部分。第一个是硬件的 **interface**，向系统以外的部分展示的就是接口而已。第二个部分就是 **internal structure**。这部分是为了给系统提供一个抽象层，里面可能有些许芯片，一些复杂的设备里甚至有简单的CPU，也可能有通用存储器或者设备专用芯片。举个例子，现代的RAID控制器可能包含成百上千行代码的 **firmware**。

![1547040838064](/img/in-post/持久性Persistence.assets/1547040838064.png)

## 36.3 规范协议

上图是一个最简单的device，interface包含3个寄存器，一个 **status register** 用于读取当前设备的状态；一个 **command register** 用于告诉设备执行一个特定任务；一个 **data register** 用于从设备输入或输出数据。通过读写这些寄存器，OS可以控制设备的行为。

以下面的简单例子描述一次和OS的互动：

![1547041731780](/img/in-post/持久性Persistence.assets/1547041731780.png)

## 36.4 用中断降低CPU的开销

程序一直等不现实，所以当程序执行一次I/O，OS发起了一个请求，把调用的进程休眠，然后执行另一个进程，当设备完成操作后，发起一个硬件中断，让CPU跳到OS预先定义的 **interrupt service routine (ISR)** 。这个handler就是OS写的一些代码，完成这次I/O请求并唤醒那个请求的进程。

中断利用了CPU，降低了CPU的空等开销。

但是中断并不总是最好的，有时候设备处理地非常快，当第一次poll就发现设备已经搞定了，这个时候中断就会带来频繁的切换开销。所以设备快的时候，Poll比较好；如果慢，中断可以利用CPU空挡。不过设备速度不固定的话，用混合方法，先poll一会如果没完成立马中断，这叫做：**two-phased** 方法。

不适用中断的另一个理由是网络的诞生，当很多数据包过来的时候，可能会让OS处于 **livelock** 状态，OS一直处理中断的事情了，没让任何一个 user-level process 运行。这种情况最好用polling先处理一些请求即使更多的数据包抵达。

另一个基于中断的优化是 **coalescing** 。就是一个确定的中断要产生前，等一小会让其它的请求继续处理，这样做可以把某一批的中断聚合一起做了。不过等太长也不好，要找一个 trade-off 。

## 36.5 用DMA实现效率地数据移动

不幸地，传输数据本身也要占用CPU，之前讨论都忽视了这点，如果加上数据拷贝时间，即"c"，下图：

![1547044336370](/img/in-post/持久性Persistence.assets/1547044336370.png)
解决方案就是 **Direct Memory Access (DMA)**。DMA设备可以在CPU不介入的情况协调 内存和设备 之间的传输。

OS告诉DMA data在内存的地址，拷贝多少，哪个设备。DMA完成后发起中断，OS就知道传输完成了。下面是一个简单的timeline：

![1547044537016](/img/in-post/持久性Persistence.assets/1547044537016.png)

从这个 timeline，你看得出CPU在拷贝时处于空闲可以做其它事情。

## 36.6 设备交互的方法

第一种（老）：显示地执行 **I/O instructions**。指令声明了一个发送到特定设备的路径。比如在X86中，in 和 out 指令用于和设备通信。caller需要指定寄存器，指定device的端口。

那些指令一般是特权级别的。OS控制着设备，因此OS也是唯一能和设备交互的设施。如果用户级程序能随便和硬件交互，那就有可能通过 loophole 来获取整个机器的控制权。

第二种（新）：**memory-mapped I/O** 用这个方法。如果设备是内存对应的地址，那么硬件就让设备寄存器处于可用状态，然后访问的时候，OS发起load/store 时，硬件转发这些指令到设备而不是 main memory。

后者的方法不需要增加什么新的指令，所以是最好的方法。

## 36.7 适配OS：设备驱动

OS中有一部分代码确切地知道设备工作的细节。这部分软件就是 **device driver**，它们知道设备的接口信息。下图是一个粗略的描述Linux软件的组织形式。跨越层级的时候都是透明的。

![1547530964572](/img/in-post/持久性Persistence.assets/1547530964572.png)

> Raw interface 使得某些特别的应用（file-system checker）直接读写block，大部分地系统都提供这样的接口。

所以说OS中代码大部分都是驱动，而且这些驱动代码大部分并没工作，因为你的设备不可能一次插入非常多的设备。

## 36.8 例子：简单的IDE磁盘驱动

一个IDE磁盘提供一个简单接口给系统，4种类型的寄存器：control, command block, status, error。这些寄存器通过读写专用的 ”I/O地址“ 生效。

![1547261680985](/img/in-post/持久性Persistence.assets/1547261680985.png)

**下面是基本的交互的协议：**

- Wait for drive to be ready。eg: 一直读 status register 直到设备Ready不忙的时候
- Write parameters to command registers。eg: 写扇区号、扇区的logical block address（LBA）、驱动号这些玩意儿到 command register
- Start the I/O。eg: 通过发射 R/W 到 command register。
- Data transfer (for writes)。eg: 一直等到设备状态为READY和DRQ（drive request for data）；写数据到数据端口。
- Handle interrupts。eg: 复杂点的方法是等待一批传输中断合在一起处理。
- Error handling。

**可以看到这些协议已经在xv6中实现：**

![1547262186915](/img/in-post/持久性Persistence.assets/1547262186915.png)



# 39 File & Directories

在三个主题之一虚拟化，我们知道进程是CPU的虚拟化，地址空间是内存的虚拟化。现在进入 **persistent storage**。这里的数据都是非易失性。

## 39.1 Files and Directories

- 文件很简单，就是线性的字节数组。文件通常有一个 **Low-level name**。因为历史缘故，low-level name叫做 **inode number**，后面会提到现在仅仅假设每个file都有对应的inode number。OS不知道文件的结构，不知道里面存了什么类型的东西，FS负责存储文件，确保你要用的时候找到它。

- 目录和文件很像，也有一个 inode name，但是内容包含了一个由（用户读到的名字，inode name）pair组成的列表。目录层次从 **root directory** 开始。



文件的后缀名往往是惯例，没有强制说 .c 文件里存的就是C源码。

## 39.3 Creating files

通过 `open` 系统调用，通过调用 open() 传递一个 O_CREAT 的flag，程序可以创建一个新的文件。

```
int fd = open("foo", O_CREAT | O_WRONLY | O_TRUNC);
```

第二个flag代表仅当以O WRONLY行为模式打开时才能写入；第三个flag代表如果已经存在就把之前的文件变成 0个byte的size。

返回的值叫做 **file descripter** ，就是个整数，每个进程一份私有的。

> 文件指针结构体里有一些信息，其中一个成员就是对应的文件描述符

## 39.4 读写文件

![1547520242548](/img/in-post/持久性Persistence.assets/1547520242548.png)

重定向了echo程序的输出到foo，然后把hello放进去，cat是怎么回事呢？用 **strace** (追踪系统调用)工具瞅一瞅。

![1547520372100](/img/in-post/持久性Persistence.assets/1547520372100.png)

运行的进程已经有3个文件描述符了，0：标准输入，1：标准输入，2：标准错误。所以打开一个文件分配的就是3了。read的第一个参数是fd，第二个参数是读取的buffer，第三个是buffer的size，比如4KB。

## 39.5 读写文件，但非顺序

使用 lseek() 系统调用，函数原型如下：

```
off_t lseek(int fildes, off_t offset, int whence);
```

第二个参数是偏移量，第三个决定该怎么执行这个seek。

lseek就是用来移动读写位置的。lseek()和磁盘的seek操作无关，仅仅改变了内核中的一个变量。

每个进程维护了一组 file descriptors，其中每一个对应 **open file table** 中的一个entry。每个entry追踪这个描述符打开的是谁、当前的偏移量、文件是否可读写等等。

![1547526417538](/img/in-post/持久性Persistence.assets/1547526417538.png)

对于进程打开的每一个文件，OS追踪一个 "current" 偏移，这个决定了文件中读写的位置。有两个方式修改这个current变量，一种是通过读写N个字节，然后current+N；第二种是lseek显示地直接更改。这个偏移量放在 struct file 里，process 结构里可以引用到它。

file structures 通过一个 **open file table** 来索引，xv6把这些组织成一个数组，每一个entry一个lock。

![1547526687066](/img/in-post/持久性Persistence.assets/1547526687066.png)

上图就是一个例子，一个进程打开了两次文件，那么它们分别有一个偏移量记录。记录在对应的array中。

## 39.6 Shared File Table entry: fork() 和 dup()

一般情况下，文件描述符和 open file table 的入口一一映射，但是有特例。

在某些时候，一个entry可能是 shared。父进程通过fork创建一个子进程。写一个例子程序，子进程 lseek 改变偏移，然后看父子进程的偏移在哪里，结果显示都被改变了！下图就是真实情况的样子，当一个entry被引用了，**referenct count** 增加1，当所有进程都关闭了这个文件，这个entry才会删除。在**父子进程**关系下可以共享entry很有必要。



![1547539102711](/img/in-post/持久性Persistence.assets/1547539102711.png)

另一个共享entry的例子是 dup()。这个调用允许进程创建一个文件描述符引用到已经打开的一个描述符上。

## 39.7 用 fsync() 立即写

之前write的时候只是给OS一个请求，可能会过一会才真正的发起I/O操作。但如果想立刻写入，比如DBMS需要这个操作，用额外的控制API叫做 fsync( int fd )。这个操作强制把 **dirty data (还没写入的)**写入到磁盘。当写入完成后，fsync才返回。

一个使用例子：

![1547523083290](/img/in-post/持久性Persistence.assets/1547523083290.png)

## 39.8 重命名文件

mv foo bar 用命令行可以重命名，追踪系统调用，它执行了 rename( char *old, char *new )

这个调用是原子操作，以应对系统的崩溃。

这个在编辑文档时也很有用。下面的例子中，当你增加内容，写到一个新的临时文件，然后立即写入磁盘，最后通过 rename 交换两个版本，并删除旧版本。

![1547525741135](/img/in-post/持久性Persistence.assets/1547525741135.png)

## 39.9 获取文件信息

除了文件访问，希望文件系统记录文件的有关信息。这样的data叫做 **metadata**。使用 stat() 调用可以直接看到文件的metadata。这个调用填充如下的数据结构：

![1547539681306](/img/in-post/持久性Persistence.assets/1547539681306.png)

这样的信息我们叫做 **inode**，至今你先把它想成FS维护的一个和某个文件有关的数据结构。inodes一般在外存，但是一部分copy会缓存到memory以加速访问。

## 39.10 删除文件

在命令行删除文件是 `rm`，但是rm的时候相关的系统调用是 `unlink()`，为了理解这个需要学习目录。

## 39.11 创建目录

用户不能直接写入信息到目录，因为目录的格式是FS metadata。你只能间接地更新一个目录（比如创建目录，创建文件等）这样FS才能保证目录地信息是期望地格式。

当目录创建后，虽然空，但是还是有一些最小内容。一个entry指向它自己，一个entry指向它的父辈。

## 39.12 读取目录

![1547540950454](/img/in-post/持久性Persistence.assets/1547540950454.png)

写一个简单的程序可以打印当前目录里的内容，仅仅用循环打印 inode number 和 name。

每一个目录的entry里面的信息：

![1547541023487](/img/in-post/持久性Persistence.assets/1547541023487.png)

因为目录里没有存什么信息，程序有可能调用 `stat()` 来获取更多的其它信息。

## 39.13 删除目录

调用 `rmdir()` 删除一个空的目录，非空的话报错。

## 39.14 硬链接

之前谈到，删除一个文件执行 `unlink()` 调用。那么反过来就有一个 `link()`，它接收两个参数，old pathname 和 new pathname，新的文件名指向旧的那个。

![1547545764729](/img/in-post/持久性Persistence.assets/1547545764729.png)

说明下，link 仅仅创建另一个pathname，指向了相同的 inode number，文件本身没有复制一份。通过ls -i可以看出来：

![1547545827768](/img/in-post/持久性Persistence.assets/1547545827768.png)

通过传递 `-i` flag 给 ls，打印出 inode number。link出来的Old 和 new 都指向了同一个 inode struct，只是 readable name 不同。

当你 rm 删除一个硬链接的文件时，另外一个名字还是可以用，文件没删除？因为unlink的时候它去检查 **reference count** （有时候叫 **link count**）允许文件系统追踪有多少通过的 file names 链接到了这个 inode。只有inode中 link count 为0时，才会真正的释放inode和相关的 data blocks，这才释放了文件。

用 `stat()` 可以看出来相关的信息：

![1547546364698](/img/in-post/持久性Persistence.assets/1547546364698.png)

## 39.15 符号链接

硬链接的相反就是软链接（**soft link**）。硬链接有限制，你不能链接到一个目录；不能在不同的分区下链接，因为inode number在同一个文件系统下才有效。

创建这样的链接要用到 `ln -s`。符号链接就是那个文件自身。它只是存储了路径名字，所以符号链接生成的文件大小只和路径长度有关（有趣！）

## 39.16 权限

下层的操作系统使用各种技术以安全可靠的方式共享有限的物理资源。文件系统也提供了一个磁盘的虚拟化视角，把磁盘“好像”从一堆原始的块转变成更加用户友好的文件和目录。但是这个抽象和CPU、Memory里的那种十分不同。因为文件是人人都有可能访问到的。

需要一系列机制做权限问题。

第一个机制是 **permission bits**，看到一个文件的 permission 键入：`ls -l filename` 

![1547701750597](/img/in-post/持久性Persistence.assets/1547701750597.png)

第一个字符是 `-`，代表这是个普通的文件，不是目录(d)和符号链接(l)，后两者没有权限，所以不考虑先。从`rw-`开始每3个字符代表一个用户组的访问权限，依次是 **owner**，**group**，**others**。

ower可以轻易改变权限，举个例子，同过chmod命令。

## 39.17 制作并挂载FS

如何从这么多的文件系统组装一个完整的目录树？

用 `mkfs` 命令的 idea：

give the tool, as input, a device (such as a disk partition, e.g., /dev/sda1) and a file system type (e.g., ext3), and it simply writes an empty file system, starting with a root directory, onto that disk partition. And mkfs said, let there be a file system!

文件系统创建后需要挂载才能在同一个 文件系统树 下被访问。用 mount 命令，挂载到某一个目录上最为 **mount point**。

举个例子：有一个ext3文件系统在设备分区 sda1下，挂载到 /home/users 下：`mount -t ext3 /dev/sda1 /home/users`

假设之前有个文件foo在ext3中，现在通过/home/users/foo能访问了。输入 mount 命令看都有什么挂载了。



# 40 File System Implementation

这一章节介绍一个 **very simple file system**。文件系统和进程内存不一样，不需要额外的硬件，纯软件！

## 40.1 思考的方式

思考下面两个方面：

1. 数据结构：磁盘上用什么类型的结构以方便FS来操作呢？
2. 访问过程：怎么把进程的调用，eg:open() read() write() 映射到上述的结构中。

学习FS从两个方面思考很简单！

## 40.2 整体结构

现在我们开发的一个雏形，把磁盘分割成 **blocks**，每个4KB（习惯）。现在我们就建立了一个最最最简单的FS了，一系列的 blocks而已，假设64个，如下：

![1547711688820](/img/in-post/持久性Persistence.assets/1547711688820.png)

磁盘肯定存我们的用户数据，所以大部分都给user，给user划分一个 **data region** 。保留一个固定区域。

之前说了，FS要追踪每个文件的信息，这些信息是 **metadata** 的关键部分，并且还要追踪 file 由哪些blocks组成的，文件的size，权限什么的，这些都在 **inode** 里存。

为了存储一些列的 on-disk inodes，需要在磁盘准备一个区域叫 **inode table**。下图中 i 代表 inode，D 代表 data

![1547712499955](/img/in-post/持久性Persistence.assets/1547712499955.png)

一般inode不太大，如果 inode 是 256 字节，4KB能放16个 inodes，上面的例子中就有80个Inodes（最大的文件数量）

管理blocks可以用一个 **free list**，但是这里简单点，用一个流行的结构：**bitmap**，一个给数据用，一个给inode table 用（data/inode bitmap）

![1547713012239](/img/in-post/持久性Persistence.assets/1547713012239.png)

左边的那个 block 用来当 **Superblock**。包含了FS的相关信息，比如一共多少inode blocks、多少data blocks，它们分别从哪里开始等等。

所以 mount 的时候，OS先读取 superblock。

## 40.3 文件组织：inode

每个inode由一个数组（**i-number**）指引，这就是我们之前说的 **low-level name**。

![1547713564427](/img/in-post/持久性Persistence.assets/1547713564427.png)

为了读取 inode number N 的inode，FS计算 

```
inodeTableStartAddr + N*sizeof(inode) 
```

**FBI WARNING：**磁盘不是字节寻址，而是一堆可寻址的扇区（磁盘），通常512字节。所以 20KB/512B = 40，寻址40号 sector。至此找到了inode。

![1547715449690](/img/in-post/持久性Persistence.assets/1547715449690.png)

上面的信息都是一个文件的 **metadata**。

#### Multi-Level Index

为了支持更大的文件。引入了间接指针！不过直接指针也是有的，比如12个直接指针，一个单独的间接指针，一个block是4KB大小，disk address是4B，那么可以有1024个直接指针被间接引用，这样文件最大就能到（12+1024）*4K=4144KB。

这样的一个不平衡的树形被叫做 **multi-level index** 方法。很多文件系统使用多级索引，ext2、ext3和原始的UNIX文件系统。但是 XFS和ext4使用 **extents** 而非简单的指针。它们和虚拟内存中的 **segments** 类似。

为什么使用这样的不平衡树？为什么不是其它？因为大部分的文件都是小文件，所以这样的组织能最大化优化这样的场景。

![1548082344057](/img/in-post/持久性Persistence.assets/1548082344057.png)

## 40.4 目录组织

可以采用的一个简单的方法，包含一个列表(entry name, inode number)pair。文件系统一般把目录也当成文件存储，所以目录也有 inode，在inode table里，不过类型上是“目录”而非常规文件。

XFS等文件系统使用 B-tree 的形式存储目录，那样更快。

## 40.5 空闲空间管理

在这个简单的文件系统中，我们只用两个简单的 **bitmap** 做这件事。

XFS使用 **B-tree** 来表示哪些chunk是free。

ext3等系统也不是说用一块只分出一块，而是一次找一连串的blocks，因为这样能保证文件的部分在磁盘中连续提升性能，这样的 **pre-allocation** 策略很常见。

## 40.6 Reading和Writing的路径

现在跟一遍 access 以熟悉，假设现在FS已经挂载并且superblock已经在内存中了，其它的东西还在磁盘。



**Reading a file from disk**

比如你要打开一个文件（eg: /foo/bar）读后关闭。这个文件只有12KB，也就是3 blocks。

当发起这样的一个 open call ，OS首先要找到bar的 inode，来获取基本的关于文件的信息（权限、大小等等）。不过只知道一个路径名的字符串，要先通过转换定位到 inode。所以呢，先找 `/` 目录，这个目录是已知的，除了它其它目录都要通过父母找到。大部分系统，`/`目录的 i-number 为2，所以读取对应的block，找到2的inode。然后找这个 inode 的data blocks，包含了root目录的内容。通过读取遍历一些data blocks，找到了foo的入口，就找到了foo的 i-number，下一步递归地找到最后的文件的 inode。找到后把foo的 inode 信息读入内存中，FS在 per-process open-file-table 中分配 file descriptor 给相应进程，然后返回给user。

一旦 open file，程序就可以发起 read 操作。除非用 lseek() 主动改偏移，不然都是从0偏移开始（即从第一个data block开始读），读完第一个 data block 更新对应的 open-file-table 中信息，然后下次读就是下一个data block。

关闭文件简单的多，仅仅回收 file descriptor，更新 open-file-table，甚至都不要发送任何 disk I/O。

下图就是一个对应的 timeline，每次read要更新 inode 中最后的访问时间，所以要 write inode。

![1548231090128](/img/in-post/持久性Persistence.assets/1548231090128.png)



**Writing to Disk**

写流程很类似，首先打开文件，然后发起一个 `write()`调用来写入内容，最后文件关闭。

除非是覆盖写入，不然 writing 要 **allocate** 一个block。而且写新文件时还要决定哪个block用来分配。所以逻辑上每次 write 会生成5个 I/O 操作：一个读 data bitmap，一个写 data bitmap，还有两个用来读写 inode，第五个写对应的 data block 本身。

这样看 I/O 开销很大！简单的创建文件就要很多操作：read inode bitmap（为了找到free inode）；write to inode bitmap（标记分配了）；write the new inode itself（初始化）；写目录（把文件链接到目录中）；读写目录的inode；如果目录的空间不够了，还要申请额外的directory block。

![1548245525050](/img/in-post/持久性Persistence.assets/1548245525050.png)



## 40.7 Caching & Buffering

I/O开销过于巨大，一般使用内存来缓存重要的 blocks。

传统的 **static partitioning** 有坏处，不够灵活，比如把内存中的10%区域作为文件系统的缓冲区。

现代的系统一般使用 **dynamic partitioning**，大部分OS把 virtual memory pages 和 file system pages 整合进 **unified page cache**。这种方法，内存的分配可以更加灵活。

以上都是关于 caching on reads，下面说 caching on writes。

write必须把操作发到设备上，那么这个buffering策略不太一样。FS可以批量一批更新到更少的 I/O 操作中。把writes缓存到内存的好处非常多。

但是！某些应用（数据库等）并不能因此受益，为了避免缓存导致的数据丢失，一般直接通过调用 `fsync()` ，使用 **direct I/O** 接口绕过cache。

## 40.8 小结

关于文件的信息是 metadata，通常存在 inode 中，目录就是特别的一种文件，FS使用类似bitmap的结构追踪哪些data block和inodes是free的。

当然，文件系统的设计非常自由，也没有共通的标准，下面会看到各种区别。

# 41 Locality & Fast file system

## 41.1 羸弱的性能

古老的 UNIX file system 更像把磁盘当作一个随机访问的内存，数据都是分散的。举个例子，数据块经常和它的inode很远，索引开销非常大。

比如有些文件的数据分布在几个连续的区域，这些区域相互被分割，这样的话每次都要 seek 后才能读取数据。这个问题可以用一些 **defragmentation tools** 完成。

另一个问题是：原始的 block size 太小了（512B）这样传输数据效率不高，(虽然block小，内部碎片小)

## 41.2 感知的解决方法

伯克利的FFS思想是将文件系统结构和分配策略设计为“磁盘感知”，从而提高性能。因此Fast File System开创了新纪元。保持接口一致，但改变内部实现。所以现有的文件系统都是改变了接口的内部设计。

## 41.3 组织结构: Cylinder Group

![1548302064975](/img/in-post/持久性Persistence.assets/1548302064975.png)

磁盘由许多的 **cylinder groups** 组成，比如图中的group中有3个cylinder。

因为现代磁盘不会暴露过多细节给FS，ext4仅仅把磁盘组织成 **block groups**，不管你说**cylinder groups** 还是 **block groups**，核心思想一样：把同样的文件放在同一个group中。

用groups来存放文件和目录，FFS需要追踪group的相关信息，FFS给每一个group准备了一个结构，比如inode、data blocks和其它的元数据。比如FFS中每一个单独的 cylinder group 就有下面的一个结构：

![1548314079325](/img/in-post/持久性Persistence.assets/1548314079325.png)

**super block** 是一份拷贝，每个group都有；**inode bitmap 和 data bitmap** 用来追踪具体的文件。



## 41.4 策略:如何分配文件和目录

基本思想：把相关的放在一起，把不相干的远离。FFS使用了一些简单的启发式放置方法。

第一，目录的放置。找分配比较少的目录，同时具有比较多的free inodes的group去放置目录的inode和data。

第二，文件的放置。把文件的inode和data放在一起，把文件和其目录尽量放一起。

下面的例子就是/a/c, /a/d, /a/e, /b/f 的放置情况：

![1548315950190](/img/in-post/持久性Persistence.assets/1548315950190.png)

## 41.5 测量文件局部性

![1548316061144](/img/in-post/持久性Persistence.assets/1548316061144.png)

## 41.6 大文件的例外

对于大文件可能会直接填满一个group，所以FFS这么做：先在第一个 block group 分配一些blocks（比如就是12，直接索引的数量），然后在另一个 block group 中分配同样的数量，依次类推。

比如下面的例子中，设置每个group的分配数量为5，那么a这个文件就如下面所示：

![1548336502579](/img/in-post/持久性Persistence.assets/1548336502579.png)

随着磁盘的容量越来越多，同一个 surface 更多的bits，需要在每次 seek 之间传输更多的数据才行。

## 41.7 FFS的其它事项

内部碎片的解决方案：FFS设计者引入了 **sub-blocks** ，每个是512KB，那么如果你创建了一个小文件，比如1KB，占用两个 sub-blocks 不会浪费整个 4KB block。随着文件增长，FS会持续分配小的sub-blocks，直到填满 4KB block，然后把 sub-blocks拷贝进去，释放sub-blocks以便未来使用。

上述的效率太低，实际上FFS通过修改 libc 库，以实现缓存writes，然乎凑够4KB chunks 再发送到FS。

![1548338086075](/img/in-post/持久性Persistence.assets/1548338086075.png)

第二件事就是寻道的策略，比如读完block 0，发起了一个block 1的read，这时为时已晚，磁头已经到 block 1 了，因此完全转一圈才行。FFS的解决方案很简单，把相邻的block 交替着放，这个方法叫做 **parameterization**。

你也许会想这个方法不是最棒的，因为现代的系统更加聪明，每次读入一个track的内容缓存起来，这对于FS都是透明的。这种内部的 disk cache 也叫做 **track buffer**。FS不必担心这些低级别的细节。

## 41.8 总结

FFS是一个 file system 发展的分水岭。中心思想就是：treat the disk like it's a disk。借鉴FFS的有 ext2 ext3 等等。



# 42 FSCK & Journaling

系统也许会崩溃会断电，on-disk state 可能只被部分更新，crash之后，系统启动后挂载文件系统，怎么保证文件系统维护 on-disk image 在一个可信任的状态。

## 42.1 Detailed example

![1548339483081](/img/in-post/持久性Persistence.assets/1548339483081.png)

这个例子中，要追加一个 data block 到某个文件中，上图是初始状态，追加后，在系统的内存中，我们有3个blocks（inode、iB、dB）要写入到disk，更新后的inode看起来如下：

![1548339559322](/img/in-post/持久性Persistence.assets/1548339559322.png)

最终完成后的 on-disk image 如下：

![1548339582381](/img/in-post/持久性Persistence.assets/1548339582381.png)



**The Crash Consistency Problem**

从这些场景中，可以看得出来有许多奔溃问题发生在 on-disk file system：文件系统中的数据结构的不一致性；空间的泄漏；返回垃圾数据给用户等。

我们想要一种原子操作，但是磁盘的 commits 一次一个write，然后crashes or power loss 也许会发生。这个问题就叫：**crash-consistency problem**



## Solution #1: File system checker

早期的文件系统对于 crash consistency 采用的是让不一致性发生，然后重启后再修复它的思路。但是不能修复所有情况，比如FS看起来一致的，不过 inode 指向了垃圾数据。

fsck 工具在文件系统挂载前运行，完成后FS就能一致了。下面总结FSCK工具做了什么：

- **superblock:** fsck首先检查 superblock 是不是合理。通过检查FS的size和已经分配的blocks数量对比。找到一个嫌疑的 superblock，这种情况系统就要使用 superblock 的一份拷贝。
- **Free blocks:** 通过 inode 得指针找所有分配的 blocks 和 bitmap 做对照。
- **inode state:** fsck 确认每个分配的 inode 都是一个有效的类型，比如 regular file、directory、symbolic link等，如果发现问题了，嫌疑的inode就会被fsck清理掉，同时 inode bitmap 也要更新。
- **inode links:** fsck会验证inode的链接关系，从root目录开始构建整个目录树，以查验是否符合链接的情况。
- **Duplicates:** 如果有重复的指针指向同一个block，并且其中一个inode明显是bad，就要清理掉。
- **Bad blocks:** 如果一个指针明显指向了超出它的 valie range，那么就把这个指针移除出 inode。
- **Directory checks:** 略

FSCK需要对FS的结构很了解，这太复杂。而且扫描整个磁盘太慢，随着disk  or RAID 的容量增长，fsck的性能降低得很厉害。

FSCK也有点不合理，因为要扫描整个磁盘以修复少量的内容太慢了，就好像你把钥匙掉到了地板上，但是你要找整个房间角落来搜索这个钥匙一样。

## Solution #2: Journaling（WAL）

最好的解决方案是从数据库那边偷来的，这个思想就是 **write-ahead logging**，专门解决这类问题。文件系统中，出于历史原因也叫它 **journaling**。Ext4、XFS、NTFS都是这种。

思路简单的很，更新磁盘时，就地更新数据结构之前，先写下一个小的note（已知的磁盘的某个地方）描述你将要做什么。并且把这些Notes组织成log形式。

通过这些note，就能保证如果crash发生了，可以查看你写的log然后再做一次。因此，你就知道要修复什么了，而非扫描整个磁盘。

现在描述 **ext3** 怎么做的。磁盘被分成了 block groups，每个block group包含了 inode bitmap，data bitmap，inodes和data blocks。日志占据了分区的一小部分。

![1548428052527](/img/in-post/持久性Persistence.assets/1548428052527.png)

上面就是ext2的大概样子，下面是ext3的大概绘图：

![1548428083456](/img/in-post/持久性Persistence.assets/1548428083456.png)





**Data Journaling**

下面的例子能理解 **data journaling** 是怎么工作的。这是ext3中的一个工作模式。

比如现在有一个 update，我们想要写入 inode（I[v2]），bitmap（B[v2）和 data block（Db）。再把它们写入到最终 disk 位置之前，先要写到log中，内容如下：

![1548432300565](/img/in-post/持久性Persistence.assets/1548432300565.png)

事务开头的TxB告诉我们关于这次更新的信息，各个blocks要写入的地址，**transaction identifier (TID)** 的信息；中间三个 blocks 仅仅包含blocks内容本身；最后的 TxE 是这次事务的尾部标记，也包含有 TID。（上面这种叫 **physical logging**，意思就是更新确切物理的地方；对应的相反词是 **logical logging**，用一种更加逻辑的表示法，虽然复杂，但是节省空间，比如说这次更新想把 data block Db 加到 file "x" 中）

当这次事务已经安全了，就可以把文件系统中对应的数据结构修改了，这个过程叫 **checkpointing**。（把日志中悬而未决的数据更新）为了做 checkpoint，发射 writes I[v2]、B[v2]和Db到它们的磁盘位置。如果这些 writes 成功完成，就说成功 checkpointed 了文件系统。

现在有了两个操作：

1. **Journal write：**写事务，包括一个 transaction-begin block、pending data and metadata、transaction-end block。
2. **Checkpoint：**写 Pending 数据到文件系统的最终位置。

在我们的例子中，按照顺序一个一个发射 write log 的请求，但是太慢，如果能并行一次发射5个请求是不是最好？？？但那不安全，因为顺序可能被打乱，导致了突然断电的时候如下的情况：

![1548434330712](/img/in-post/持久性Persistence.assets/1548434330712.png)

这个 transaction 看起来是有效的。但是 ？？ 是个什么鬼，恢复的时候，把？？恢复到目标地址，结果就完蛋了！特别是这个数据还是FS的关键一部分，比如 superblock，FS都可能挂载不了。

为了避免，FS两步法发射 transactional write 。首先把所有的 blocks 除了TxE一次性发射写入到 journal。当这些writes完成后，再发射 TxE block，以达到最后的安全状态。

![1548434657460](/img/in-post/持久性Persistence.assets/1548434657460.png)

> 磁盘保证512字节是原子的，意思就是要么不写，要么都写上，所以TxE设置为单独的 512B 的block，就是原子的

1. **Journal write**：除了TxE都写入到log，然后等待所有的writes完成
2. **Journal commit**：写入transaction commit block（包含TxE）到Log，等待写入完成就说commited
3. **Checkpoint**：写入update的内容到最终的 on-disk locations



**Recovery**

如果crash发生在transcation还没commit就可以跳过这次transcation。

如果发生在checkpoint之前但是commit之后，FS就可以恢复了。重启后，FS的恢复进程扫描整个 log 来找寻提交的transactions。这些transactions按照顺序被重放，等到恢复完成就可以继续接收新的请求。



**Batching Log Updates**

基本的WAL协议增加了许多磁盘开销。有些文件系统不是每次update都提交一次。而是把所有的更新都放到了一个 global transaction。比如创建两个文件在同目录时候，FS仅仅标记 in-memory inode bitmap，inodes， directory data， directory inode为dirty，然后把它们加入到 list of blocks 组成当前的transaction。当写入磁盘的时候，这个包含很多updates的 global transaction 被提交。通过缓存updates，FS可以避免过多的写入开销。



**Making The Log Finite**

Log 的size是有限的，如果一直加入transactions，那么肯定会满，可以做成环形的log，checkpoint后的要释放空间。所以我们在WAL协议上再加入一条 **Free** 步，这一步就是free掉所占用的日志空间。

1. **Journal write**：除了TxE都写入到log，然后等待所有的writes完成
2. **Journal commit**：写入transaction commit block（包含TxE）到Log，等待写入完成就说commited
3. **Checkpoint**：写入update的内容到最终的 on-disk locations
4. **Free：**通过更新 journal 来标记对应的transcation free





**Metadata Journaling**

之前的方法每次都要写两遍，特别对于顺序写入的workload难以接受。也带来了寻道seek等等开销。

ext3所用的 **data journaling** 之所以慢是因为日志写入了所有的更新数据。那么我们就可以想到一种简单的方法叫 **ordered journaling** （也叫做 **metadata journaling**）

![1548436708508](/img/in-post/持久性Persistence.assets/1548436708508.png)

更新由3个blocks组成：I[v2] B[v2]和Db，前两个和之前一样的处理。那么什么时候把 data blocks 写入到磁盘中呢？

如果先transaction，再写入data block，那么有可能指向脏数据（data 没写完就断电）。所以强制 data 先写入，保证指针永远不指向垃圾。这种规则就是 "write the pointed-to object before the object that points to it"

协议如下：

1. **Data write**：把数据写入到最终位置，等待完成
2. **Journal metadata write**：除了TxE的metadata都写入到log，然后等待所有的writes完成
3. **Journal commit**：写入transaction commit block（包含TxE）到Log，等待写入完成就说commited
4. **Checkpoint metadata**：写入update的内容到最终的 on-disk locations
5. **Free：**通过更新 journal 来标记对应的transcation free



**Tricky Case: Block Reuse**

下面的例子关于文件删除的时候：

foo目录要改变目录内容，因为目录被当作 metadata，所以foo这次commit会提交关于data的内容到Log。

![1548437362807](/img/in-post/持久性Persistence.assets/1548437362807.png)

后来foo目录被删除，一个新的文件foobar在其他地方创建，重用了这个block（1000）。

![1548437371932](/img/in-post/持久性Persistence.assets/1548437371932.png)

当crash发生后，恢复的时候把foo提交的data恢复进 block 1000，这样相当于foobar文件的内容被污染了。

解决方法：checkpointed之前都不要重用 block 即可



**Wrapping Up Journaling: A Timeline**

下图是两种 Journaling 的 timeline：

![1548437353322](/img/in-post/持久性Persistence.assets/1548437353322.png)

![1548570492491](/img/in-post/持久性Persistence.assets/1548570492491.png)



##  Solution #3: Other Approaches

略

## 42.5 总结

有许多方法解决一致性，传统用 FSCK，但是太慢了。后来现代的系统用 Journaling，恢复时间减少到了 O(size-of-the log)，加速了recovery。最常见的Journaling形式按照 metadata journaling 排序。



# 43 Log-structured File Systems

这个文件系统的动机处于以下发现：

- **System memory is growing**
- **Large performance gap between random I/O and sequential I/O**
- **Existing file systems perform poorly on many common workloads**
- **File systems are not RAID-aware**

所以这样的FS专注在写性能，尝试利用磁盘的顺序带宽。此外，频繁更新 on-disk metadata structures 也表现很好。这种FS就叫 **LFS**。当往磁盘写数据前，LFS首先缓存所有的 updates（包括metadata）在内存的一个 **segment**。当segment满了后，才会写入到disk中连续很长的一个部分，LFS从来不覆盖写现有数据，而总是把 segments 写到free locations。因为segments很大，性能达到了顶峰。

## 43.1 顺序写入disk

顺序写是LFS的核心思想：比如，先写入data block，然后写入inode，同时inode标记data在哪里。

![1548571274634](/img/in-post/持久性Persistence.assets/1548571274634.png)

## 43.2 效率地顺序写

因为磁盘的物理结构，如果写一个block就commit到磁盘一次，太慢，因为磁头可能那会已经转过了那个位置，要等一个周期才转回来。可以准备一个buffer，这个技术叫 **write buffering**。写disk之前，LFS追踪一堆updates在内存，足够数量后才一次写disk。这个大的chunk of updates叫 **segment** (和内存管理那个不一样)

下图的例子，第一个update是写文件 j 的4个block，第二个Update是写文件 k。每个segment真实大小大概有几MB。

![1548571887448](/img/in-post/持久性Persistence.assets/1548571887448.png)

## 43.3 Buffer多少才好？

1. 寻址时间
2. 纯访存速度
3. 效率比

![1548572478739](/img/in-post/持久性Persistence.assets/1548572478739.png)

3个因素决定每次缓存多少才好。

## 43.5 inode map

最新的inode不像之前的文件系统通过首地址+偏移算出来。需要一个inode map (**imap**)来定位最新的inode。这个imap的片段（每个entry 4B，磁盘的指针大小）一般放在最新数据的右边，如下图所示：

![1548573063174](/img/in-post/持久性Persistence.assets/1548573063174.png)

## 43.6 The checkpoint region

需要知道 imap 在哪里存放，所以 **checkpoint region(CR)** 就是干这个的，CR是周期性更新，非实时。下图就是例子：

![1548573416817](/img/in-post/持久性Persistence.assets/1548573416817.png)

## 43.8 目录怎么办

当创建一个文件foo在一个新目录时，disk结构如下：
![1548575438334](/img/in-post/持久性Persistence.assets/1548575438334.png)

访问的时候很简单，找到目录data，找到foo的number k，然后imap找k的地址，OK。

以后更新这个file只会在后面追加这个文件的内容，目录不用动， 因为目录和文件通过一层 imap 转换，只要imap记录着foo的最新地址就好。

## 43.9 垃圾回收

LFS cleaner不是直接把死数据free掉，那样会留有很多空洞，影响性能。相反，采用 **segment-by-segment** basis，为随后的writing清理出大段的空间。周期性地，cleaner检查M个old segments，然后把Live block读出来放到新的 segments，这样就有N个新的segments，N<M。拷贝后，旧的segments清空掉。

两个问题：1.怎么知道谁Live；2.多久清理一次，选哪些segments清理。



**问题1：决定Block是否存活？**

segment summary block纪录存活信息



**问题2：选那些blocks清理，何时？**

冷热segment区分对待，参考“Design and Implementation of the Log-structured File System” by Mendel Rosenblum and John Ousterhout. SOSP ’91, Pacific Grove, CA, October 1991

上述的方法不太好，最好是这样的：“Improving the Performance of Log-structured File Systems with Adaptive Methods” by Jeanna Neefe Matthews, Drew Roselli, Adam M. Costello, Randolph Y. Wang, ThomasE. Anderson. SOSP 1997, pages 238-251, October, Saint Malo, France. A more recent paper detailing better policies for cleaning in LFS.

## 43.12 恢复

crash两种情况，1.写CR时；2.写segment时

第一种情况，为了保证CR是原子操作，准备一个CR副本，同时写CR的第一个block和最后一个block时加上timestamp，如果后面的时间不是在前面的之后，那么说明发生了不一致，用副本还原。

第二种情况，因为每三十秒更新CR，数据可能十分旧了。使用 **roll forward** 技术，从最新的CR开始找 log 的end，然后依次读后面的segment，看数据是否有效，如果是就恢复，不是就算了。

参考“Design and Implementation of the Log-structured File System” by Mendel Rosenblum.http://www.eecs.berkeley.edu/Pubs/TechRpts/1992/CSD-92-696.pdf. The award-winning dissertation about LFS, with many of the details missing from the paper

## 43.13 总结

LFS的方法创新在更新磁盘，不是就地更新，而是追加数据，然后再把旧的数据回收以清理。大量的写操作相比传统文件系统能给一个更好的性能。LFS也许适合SSD。

虽然用 copy-on-write 来处理旧数据已经成为现代文件系统主流选择，但依然备受争议。

# 44 Flash-based SSD

这里说的 flash 主要指的是 **Nand-based flash** ，举个例子，写一个特定的块（ **a flash page**），你必须先擦除更大的一块（**a flash block**）；而且频繁地写一个Page会导致磨损坏掉。

> 本章说的page、block不同于之前文件系统中的。

## 44.2 From bits to Banks/Planes

只存几个bit肯定不是一个合格的外存，因此SSD被组织成 **banks** or **planes**。它们包含许许多多的cells。

**erase blocks**通常128、256KB，**pages** 通常4KB，每一个 bank 里有许多 blocks，其中又有很多 pages。

![1547778561926](/img/in-post/持久性Persistence.assets/1547778561926.png)

## 44.4 Basic Flash Operations

三个low-level操作：

- **Read a page**：flash chip的客户端可以读任何的一个Page，简单发送read和page number即可。和之前请求的位置、这个page的位置无关，这个操作一般都 10 微妙 。说明SSD是 **random access** 设备。
- **Erase a block**：写一个page，需要先擦除所在block，这就等同于把block中所有bit设为1；因此擦除前必须让block的其它有效数据拷贝到其它地方。整个擦除要几毫秒，**擦除完成后所有page都是可以programmed。**
- **Program a page**：把一个Page中的部分 1 编码成 0。比擦除动作耗时短，不过还是有 100 微妙左右。

下面的例子会告诉你一个block各个状态的转换：

![1547778005607](/img/in-post/持久性Persistence.assets/1547778005607.png)

但是当一个block写满后（每个page都是 valid）只能通过erase操作了，问题来了！其它的page怎么办？

## 44.5 From raw flash to flash-based ssd

![1547778982199](/img/in-post/持久性Persistence.assets/1547778982199.png)

SSD内部应该有一些控制设备，如上图。FTL（**flash translation layer**）能够把对 logical blocks 的读写转换成 low-level 的那三个指令以对 physical blocks、pages 操作。

加强SSD的性能可以从很多关键的技术入手。第一，利用多个 chips 的 **parallel**；第二，减少写放大（**write amplification**），写放大=FTL发送给闪存芯片总写入流量 / 客户端发送给SSD的写入流量。

加强SSD的可靠性也可以从多个点入手。比如，**wear out**，如果单独的block被擦除和编程过多次，就会变得不可用；FTL应该努力让writes尽可能发到各个blocks上，这个操作就是 **wear leveling**。

## 44.7 A Log-structured FTL

直接映射逻辑块和物理块肯定不行，现代FTL多用 **log structured**。这个思想在 device 和 file system 都很有用。一个对 logical block N 的写操作，设备把这个写操作追加到 current-being-written-to block 中的下一个空闲的 spot 上。这个行为就是**logging**。为了实现读，设备在NVM中维护一个 **mapping table**，记录 logical block 的物理地址。

**logical block addresses** 被SSD的客户（file system）记录信息存在哪里。

假设 client 发送了如下的操作：

![1547780564345](/img/in-post/持久性Persistence.assets/1547780564345.png)

第一次write接收后，SSD决定把它写到 block 0，但它是invalid，所以先erase，再program。大部分的SSD都是从低往高选page。FTL必须记录这次program的转换，在 mapping table 中追踪这个状态。

![1547780774007](/img/in-post/持久性Persistence.assets/1547780774007.png)

![1547780791842](/img/in-post/持久性Persistence.assets/1547780791842.png)



![1547780802615](/img/in-post/持久性Persistence.assets/1547780802615.png)

缺点也是有的，过度地写一个 logical block 会带来 **garbage**。设备要周期执行 **garbage collection (GC)**。

## 44.8 Garbage Collection

举个例子来讲这一节。

之前地100和101逻辑块被又一次写入，那么就把它们写到新的一个物理块上，如下：

![1547782467440](/img/in-post/持久性Persistence.assets/1547782467440.png)

垃圾回收做的就是把一个物理块中的live data读出来放其它地方，然后清理这个block。我们的例子中做如下：

![1547782536094](/img/in-post/持久性Persistence.assets/1547782536094.png)

![1547782543928](/img/in-post/持久性Persistence.assets/1547782543928.png)

垃圾回收对 live data 进行了重复的 read write，带来了开销。所以最好的情况就是一个block中的数据都是 dead，这样直接erase。

为了降低GC影响，SSD一般 **overprovision** the device；通过加入额外的空间，清理工作就能被延后放到后台处理，比如当设备不繁忙的时候。

**附：理解一个API：TRIM**

通知设备由某个地址指明的blocks不再需要了，SSD后面可以从FTL中清楚这个信息，然后在GC的时候再回收这部分空间。

## 44.9 Mapping Table Size

mapping talbe 的空间也是个问题，假设4KB的Page的SSD一共1TB，4B作为一个entry，需要1GB的空间来放mapping table，太大了，**page level** FTL的解决不太现实。



**Block-based Mapping**

直观地解决size问题就是每个block才配一个指针。不过这样带来一个问题，如果写一小点，就要拷贝整个block，带来了写放大问题。

解释一下，假设下面的例子，4个逻辑块（2000,2001,2002,2003）都映射到500的物理Block中。

![1547793630634](/img/in-post/持久性Persistence.assets/1547793630634.png)

那么如果改变 2002 的值，就需要把另外3个也拷贝出来，因为只能记录到500->8，所以映射在500的page都是dead data了。

![1547793763385](/img/in-post/持久性Persistence.assets/1547793763385.png)



**Hybrid Mapping**

这个技术，FTL让一小部分blocks处于被擦除状态并且直接把写的东西都发给它们；它们叫 **log blocks**。FTL就能在这些 log blocks 直接写任何page到log block 的任意地方而不用先复制出来。这些 log blocks 是Per-page mapping (log table)。

另一类是的Mapping（data table）是per-blocks。

查询逻辑块的时候，FTL先查询 log table，如果逻辑块的位置没找到，去 data table找。

为了让 log blocks 不要太多，FTL定期检查 log blocks（一个page有一个指针）并将它们切换成只能由一个 block 指针指向的 block。

举个例子，我们说 FTL 的逻辑块1000、1001、1002、1003放到了物理块2上，对应如下：

![1547879338159](/img/in-post/持久性Persistence.assets/1547879338159.png)

然后这些page都被 overwrite 了，那么可能就变成下面这样了

![1547879588577](/img/in-post/持久性Persistence.assets/1547879588577.png)

FTL 可以执行 **switch merge** 程序。就是把 block 0,1,2,3 变成一个 block pointer 指引的，旧的 data block 被erase成一个 log block。最好的情况下（比如我们这个例子），所有的 per-page pointers 被替换为一个 single block pointer。

![1547880179574](/img/in-post/持久性Persistence.assets/1547880179574.png)

事情不可能总一帆风顺，比如当你只写了两个逻辑块。FTL需要执行 **partial merge** 操作，以把剩下的 c d 也复制过来追加到log中。结果就是写放大（write amplification）。

最坏的情况是执行 **full merge**。这个操作需要更多的工作，FTL必须从许多其他的物理块中找page来完成这个操作。比如block 0,4,8被写到log block了，如果要把它们变成 block-mapped page。FTL首先准备data block，然后把逻辑块0放进去，再从其它地方找回来 1，2，3 把它们3个和逻辑块0放到一起。对于 0,4,8同理。这样的写放大非常大。



**Page Mapping Plus Caching**

鉴于上述的 hybrid 方法太复杂，有人提出了简单的方法减少page-mapped的开销。这个方法就是仅仅把FTL的活动部分缓存到memory，因此减少所需的内存。



## 44.10 Wear Leveling

后台一般执行 wear leveling 操作，正如上面提到的，对于长期不改变的数据，FTL 会周期读出来放到其它地方。


