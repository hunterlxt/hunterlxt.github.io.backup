---
layout:     post
title:      "深入浅出全面解析RDMA（学习笔记）"
subtitle:   "Introduction to RDMA"
date:       2019-01-01
author:     "TXXT"
catalog:    true
tags:
    - RDMA
    - Network
---

原文：[深入浅出全面解析RDMA by围城](https://zhuanlan.zhihu.com/p/37669618)

## 1 Intruduction

![img](/img/in-post/RDMA解析.assets/v2-f9a6d1f975976d7313696b1b82bfbaa5_hd.jpg)

传统的方式中，数据需要通过用户空间发送到另一个用户空间。首先从用户空间的 buffer 复制到内核空间的 socket buffer 中，然后在内核空间进行数据包封装，通过一系列多层网络协议处理，然后处理后的数据包进入NIC的buffer中。然后到另一台机器中又经过多层的网络解析，最后数据到了用户空间buffer，进行上下文切换用户程序被调用，一次传输就算完成。

**网络通信** 的现状是什么呢？有两个衡量标准，**带宽** 和 **延迟** 。延迟又分为处理阶段的延迟和传输阶段的延迟。

消息通信主要分2类：Large msg 和 Small msg，大消息中网络传输占据通信主要位置，小消息中处理开销占据通信主要地位。

传统的 TCP/IP 网络是通过内核发送消息，通过内核会导致很低的性能和很低的灵活性。性能低下的原因就是由于网络通过内核传递，导致高数据移动开销。所有网络通信协议通过内核传递，很难支持新的网络协议和新的消息通信协议和发送接收接口。



## 2 Related Work

### 2.1 TCP Offloading Engine

由于CPU需要繁重的封装网络数据包协议，为了将占用主机CPU的资源释放出来，发明了TOE技术，将上述工作转移到网卡上。这种技术需要特定的网络接口和网卡进行操作。

### 2.2 User-Net Networking

将协议处理放到用户空间去做，避免数据移动开销，从数据通信路径中彻底删除内核。U-Net的虚拟NI为每个进程提供了拥有网络接口的错觉，内核只负责链接。现在程序通过MUX直接访问网络，不需要数据复制到内核空间。



## 3 RDMA Explanation

远程直接访问内存技术，为了解决网络传输中服务器端数据处理的延迟而产生的。RDMA通过网络把资料直接传入计算机的存储区域，而不对操作系统造成影响，这样就不需要多少计算机的处理功能。

![img](/img/in-post/RDMA解析.assets/rdma2.jpg)

### 两种基本操作

1. Memory verbs：包括RDMA read、write和atomic操作。这些操作指定远程地址进行操作并且绕过接受者的CPU。
2. Messaging verbs：包括RDMA send、receive操作。这些动作涉及接收者的CPU，发送的数据被写入由响应者的CPU先前发布的为接收指定的地址。

RDMA传输分为可靠和不可靠的，并且可以连接和不连接的。凭借可靠的传输，NIC使用确认来保证消息的按序传送，不可靠的传输无法提供该保证。

### 三种不同的硬件实现

分别是 **infiniBand** 、 **iWarp** 、 **RoCE** 这三类RDMA网络。

![1545493526801](/img/in-post/RDMA解析.assets/1545493526801.png)

第一种专为RDMA设计的网络，从硬件级别保证可靠传输。RoCE v2是以太网TCP/IP协议中的UDP层实现，从性能上来看，很明显infiniband网络最好，但是网卡和交换机价格昂贵，RoCE v2和iWarp仅使用特殊的网卡即可。

因此，infiniBand技术需要对应的NIC和交换机；但是RoCE较低的网络标头是以太网标头，较高的网络标头是infiniBand标头，支持在以太网基础设施上使用，只有网卡特殊即可。

![img](/img/in-post/RDMA解析.assets/v2-06d2a7202c8d485956514bd2e6dde19d_hd.jpg)

![img](/img/in-post/RDMA解析.assets/v2-44d514dff41e8f37498d9639fa0c514d_hd.jpg)

### RDMA技术

RDMA有专用的 verbs interface 而不是传统的 TCP/IP Socket interface。使用RDMA首先建立从RDMA到应用程序内存的数据路径，可以通过RDMA专有的 verbs interface 建立，建立完成后可以直接访问用户空间的 buffer。

![img](/img/in-post/RDMA解析.assets/v2-6ce1b2d932655bfc0c4943c610b03f0b_hd.jpg)



### RDMA架构

![img](/img/in-post/RDMA解析.assets/v2-0437d8c732835402516a5202ae83686a_hd.jpg)

RDMA在用户空间提供了一系列的 verbs interface 接口操作RDMA硬件。RDMA绕过内核直接从用户空间访问RDMA网卡。RNIC的 cached PTE（Page Table Entry）就是用来将虚拟页面映射到相应的物理页面。

### RDMA技术详解

工作流程如下：

1. 当应用执行RDMA读或写请求，不执行数据复制，内核不参与，请求直接发送到本地NIC
2. NIC读取缓冲的内容，并通过网络传送到远程NIC
3. 网络上传输的RDMA消息包括 目标虚拟地址、内存钥匙、数据本身。请求既可以完全在用户空间处理（轮询排列方式），又或者应用睡眠到请求完成后通过系统中断处理。
4. 目标NIC确认内存钥匙，直接将数据写入应用缓存，用于操作的远程虚拟内存地址包含在RDMA信息。

### 操作细节

![img](/img/in-post/RDMA解析.assets/v2-0eaec505e01c7fa001c778427e50da06_hd.jpg)

RDMA提供了基于消息队列的点对点通信，每个应用都可以直接获取自己的消息，无需操作系统和协议栈的介入。

消息服务建立在通信双方本端和远端应用之间创建的Channel-IO连接之上。当应用需要通信时，就会创建一条Channel连接，每条Channel的首尾端点是两对Queue Pairs（QP）。每对QP由Send Queue（SQ）和Receive Queue（RQ）构成，这些队列中管理着各种类型的消息。QP会被映射到应用的虚拟地址空间，使得应用直接通过它访问RNIC网卡。除了QP描述的两种基本队列之外，RDMA还提供一种队列Complete Queue（CQ），CQ用来知会用户WQ上的消息已经被处理完。

RDMA提供了一套软件传输接口，方便用户创建传输请求Work Request(WR），WR中描述了应用希望传输到Channel对端的消息内容，WR通知QP中的某个队列Work Queue(WQ)。在WQ中，用户的WR被转化为Work Queue Element（WQE）的格式，等待RNIC的异步调度解析，并从WQE指向的Buffer中拿到真正的消息发送到Channel对端。

**RDAM单边操作 (RDMA READ)**

READ和WRITE是单边操作，只需要本端明确信息的源和目的地址，远端应用不必感知此次通信，数据的读或写都通过RDMA在RNIC与应用Buffer之间完成，再由远端RNIC封装成消息返回到本端。

对于单边操作，以存储网络环境下的存储为例，数据的流程如下：
1. 首先A、B建立连接，QP已经创建并且初始化。
2. 数据被存档在B的buffer地址VB，注意VB应该提前注册到B的RNIC (并且它是一个Memory Region) ，并拿到返回的local key，相当于RDMA操作这块buffer的权限。
3. B把数据地址VB，key封装到专用的报文传送到A，这相当于B把数据buffer的操作权交给了A。同时B在它的WQ中注册进一个WR，以用于接收数据传输的A返回的状态。
4. A在收到B的送过来的数据VB和R_key后，RNIC会把它们连同自身存储地址VA到封装RDMA READ请求，将这个消息请求发送给B，这个过程A、B两端不需要任何软件参与，就可以将B的数据存储到A的VA虚拟地址。
5. A在存储完成后，会向B返回整个数据传输的状态信息。

单边操作传输方式是RDMA与传统网络传输的最大不同，只需提供直接访问远程的虚拟地址，无须远程应用的参与其中，这种方式适用于批量数据传输。

**RDMA 单边操作 (RDMA WRITE)**

对于单边操作，以存储网络环境下的存储为例，数据的流程如下：
1. 首先A、B建立连接，QP已经创建并且初始化。
2. 数据remote目标存储buffer地址VB，注意VB应该提前注册到B的RNIC(并且它是一个Memory Region)，并拿到返回的local key，相当于RDMA操作这块buffer的权限。
3. B把数据地址VB，key封装到专用的报文传送到A，这相当于B把数据buffer的操作权交给了A。同时B在它的WQ中注册进一个WR，以用于接收数据传输的A返回的状态。
4. A在收到B的送过来的数据VB和R_key后，RNIC会把它们连同自身发送地址VA到封装RDMA WRITE请求，这个过程A、B两端不需要任何软件参与，就可以将A的数据发送到B的VB虚拟地址。
5. A在发送数据完成后，会向B返回整个数据传输的状态信息。
单边操作传输方式是RDMA与传统网络传输的最大不同，只需提供直接访问远程的虚拟地址，无须远程应用的参与其中，这种方式适用于批量数据传输。

**RDMA 双边操作 (RDMA SEND/RECEIVE)**

RDMA中SEND/RECEIVE是双边操作，即必须要远端的应用感知参与才能完成收发。在实际中，SEND/RECEIVE多用于连接控制类报文，而数据报文多是通过READ/WRITE来完成的。
对于双边操作为例，主机A向主机B(下面简称A、B)发送数据的流程如下：
1. 首先，A和B都要创建并初始化好各自的QP，CQ
2. A和B分别向自己的WQ中注册WQE，对于A，WQ=SQ，WQE描述指向一个等到被发送的数据；对于B，WQ=RQ，WQE描述指向一块用于存储数据的Buffer。
3. A的RNIC异步调度轮到A的WQE，解析到这是一个SEND消息，从Buffer中直接向B发出数据。数据流到达B的RNIC后，B的WQE被消耗，并把数据直接存储到WQE指向的存储位置。
4. AB通信完成后，A的CQ中会产生一个完成消息CQE表示发送完成。与此同时，B的CQ中也会产生一个完成消息表示接收完成。每个WQ中WQE的处理完成都会产生一个CQE。
双边操作与传统网络的底层Buffer Pool类似，收发双方的参与过程并无差别，区别在零拷贝、Kernel Bypass，实际上对于RDMA，这是一种复杂的消息传输模式，多用于传输短的控制消息。