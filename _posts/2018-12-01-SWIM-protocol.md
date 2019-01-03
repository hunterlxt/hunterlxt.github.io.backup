---
layout:     post
title:      "SWIM: The scalable membership protocol"
subtitle:   "一个适合大规模系统的成员检测协议"
date:       2018-12-01
author:     "TXXT"
tags:
    - Distributed System
---
SWIM: The scalable membership protocol

## 1 What is Membership ???
SWIM是一个著名的分布式系统成员协议，它基于Gossip发展而来，现在已经在P2P文件共享、分布式数据库中广泛应用。介绍这些概念前有必要说一下Membership的基本概念。
在分布式系统中，你可以想象有成千上万的members，当然你可以认为每个member就是一个process，它们通过网络互联。但是这个网络是动态的系统，随时都有member退出网络，也有新的加入网络。Membership协议只想解决一个简单的问题：**server想知道哪些是live peers**
![](/img/in-post/SWIM%20The%20scalable%20membership%20protocol/bear_sketch@2x.png)

这个问题看似简单，但是我们做系统的就要考虑到所有可能的情况：不可靠的网络可能会丢包、延迟、网络阻塞。所以这个问题就变得有趣了起来。

## 2 SWIM的特点
* Scalable：适用于大型网络，网络中有成千上万的节点都没关系，因为SWIM有减少消息传输开销的方法。
* Weakly Consistent：弱一致性说明每个node的membership list都可能不一样，但是它们总是不断地趋向于一致。
* Infection Style：又叫做Gossip，好比人群中的传染病，每个node都和某个集合的peers交换信息，最终消息传播至整个网络中去。

## 3 为什么SWIM能在大型网络中如此快速
在SWIM之前，大家用Gossip protocol来实现Membership protocol，大致流程如下：每个节点周期地发送 `hearbeat` 给网络中的所有可达节点，告诉大家它是活着的，如果在预定的设限内，它没能发出去这个信息，大家就认为它死了。这个方法保证了任何的faulty节点都能被non-faulty节点检测到。
不过每一周期网络中需要传输O(N)的数据量，这对于大型网络几乎不可能。而SWIM放宽了failure detection的要求。

## 4 SWIM怎么做的
说了这么多废话，SWIM在每个周期的检测流程如下：一个node A发送`ping`给list中的随机一个node（比如就叫B吧），如果B收到了就返回`ack`给A。但是如果A没在预定的时间（小于周期T）内收到这个`ack`，它会在list中随机挑选 k 个node并发送 `ping-req(B)`请求它们帮助自己来确认B是否活着，若没有任何一个node告诉A说B活着，那A认为B死了，并把它从list中移除，然后广播到整个网络中去。当然最好的情况是它没死，别人替它返回了 `ack` ，示意图如下：
![](/img/in-post/SWIM%20The%20scalable%20membership%20protocol/bear_sketch@2x.png)

> 总结一下它的做法：这里面有两个关键参数，即协议的周期时间T和随机选择的子集大小k。上图的整个过程都发生在一个T以内。SWIM的精妙之处在于它利用了网络中的其它节点帮它发送消息，避免了A和B之间的网络拥塞。  

PS：这个协议看样子还是比较简单的，在模拟的网络环境中实现了一个