---
layout:     post
title:      "CAP理论和最终一致性"
subtitle:   "CAP theorem and Eventually Consistency"
date:       2019-01-03
author:     "TXXT"
catalog:    true
tags:
    - Distributed System
---

>说明：本学习笔记主要参考伊利诺伊大学云计算课程和2008年Werner Vogels发布的文章

# 1 CAP理论

CAP的结论非常简单：在分布式系统里，有3个属性非常重要，但只能同时满足其中的2个。

1. Consistency：all nodes在任何时刻看到的data都是一样的（或说client的read操作总是返回最新写入的那个value）

2. Availability：系统时刻都允许操作，并且操作总会快速被Coordinator响应，最终client很快就得到返回的结果
3. Partition-tolerance：尽管网路有时候会因为故障导致被分隔开，但是系统依然在正常工作（或者说在满足前述的条件下工作）



### 为什么这三者如此重要？

**For Availability：**

经统计，对于Google、Amazon这样的数据公司，系统增加500ms的延迟就会损失公司20%的收益，所以必须快速且可靠地进行 Read/Write。

**For Consistency：**

比如在银行系统，任何client都必须查看到最新的updated data item，不然就会给交易造成致命的影响。

**For Partition-tolerance：**

因为Internet可能因为某些原因随时断开（Router故障、海地线缆断开、DNS故障）；即使在同一个data center里，故障都会随时随地地发生，比如 Rack switcher 宕机。



### CAP权衡

如今的云计算环境里，因为网络随时都会被隔离开来，这是无法避免的，P是必须满足的，那么CAP暗示一个system要在C和A中做出抉择。

比如，Cassandra就选择了AP，对于C只能保证 Eventual Consistency（弱一致性）；传统的RDBMS在一个partition里保证可用性。

CAP三角形的权衡图如下所示：

![PNG 图像](/img/in-post/CAP理论和最终一致性.assets/PNG 图像.png)

## 2 The Mystery of 'X'

client做每次操作时（Read/Write）可以选择一个consistency level：

ANY：any server（may not be replica）这种方式是最快的，coordinator把writes缓存起来并且快速回复给client，即使根本就没把这个操作给任何node实际处理。

ALL：all replicas，确保强一致性，但是最慢的（返回给client操作完成时，所有的replica的一致性都通过了）

ONE：有一个replica返回了ACK给coordinator，那么就算完成操作（比ALL快）。

QUORUM：用法定数量的ACKs来确认操作的完成。

### Quorum（法定数量）

![PNG 图像-1539914091312](/img/in-post/CAP理论和最终一致性.assets/PNG 图像-1539914091312.png)

比如上图这个例子里，Quorum = majority > 50%，该key-value对应了5个node。Client 1 write的时候选择了红色的3个node写入；随后，Client 2 read的时候选择了蓝色的3个node，至少有一个node包含最新的写入。因此，Quorum比ALL更快，但仍确保了强一致性。

很多k-v store的 X 都用的是quorum。

**Read流程**

1. client声明一个 R = read consistency level
2. Coordinator 等待R个replica响应
3. 在后台，Coordinator检查（N-R）replicas，如果需要就进行 repair

**Write流程**

1. client 声明 W = write consistency level

2. client把这个 new value 写到w个replica后返回

   对于step 2，分2种情况：A.同步（coordinator blocks until quorum is reached）B.异步（Just write and return）

**two necessary conditions：**

W+R > N		W > N/2

**不同情况分析**

W=1,R=1：write and read都非常少

W=N,R=1：great for read-heavy workloads

W=N/2 + 1,R=N/2 + 1：great for write-heavy workloads

W=1,R=N：great for write-heavy workloads



## 3 客户端和服务端视角下的一致性

有两种视角看待一致性，第一种是从开发者的角度，开发应用的人如何观察数据的更新，另一种是服务端的视角，更新的内容如何在系统种流通，以及系统怎么保证这些更新。

### Client-side Consistency

客户端包含下述部分：

* **A storage system.** 现在把它当作黑盒，假设它是一种大规模且分布式的东西，为了保证持久性和可用性而生。
* **Process A.** 这个进程用于读写上面的storage system。
* **Processes B and C.** 这两个进程完全独立于进程A，也读写存储系统。他们相互独立且需要互相联系。
* **Strong consistency.** 一旦更新操作完成，任何后续的访问都是最新的值。
* **Weak consistency.** 系统并不保证后续的访问总是能访问到最新的值，从更新开始到保证任何观察者看到更新值的这段时间被称为不一致窗口。
* **Eventual consistency.** 弱一致性的一种特殊形式；存储系统保证如果再也没有新的update，那么最终所有的访问将会返回最新更改的updated value。在没有故障的前提下，不一致窗口期由下列因素确定：通信延迟、系统负载、涉及的副本数量。最常见的实现最终一致性的系统是DNS，在DNS中，一个域名的更新操作将结合有过期机制的缓存被分发出去，最终所有的客户端都能看到最新的值。

### Server-side Consistency

首先明确几个定义：

W = write的法定ack数量

R = read的法定ack数量

N = 存储数据对应replica的node数量

如果W + R > N，那么R和W的场景总是有交集，就能保证强一致性。但是有时候 W + R <= N，这个时候想要保证一致性，一般通过使用延迟的方法，在后台系统会把更新传递到其它的node中去。



（今天先写到这里吧，困了，睡觉）

#分布式理论