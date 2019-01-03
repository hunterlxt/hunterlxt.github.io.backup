---
layout:     post
title:      "布隆过滤器"
subtitle:   "Bloom Filter"
date:       2018-09-15
author:     "TXXT"
catalog:    true
tags:
    - Algorithm & DS
---

## 1 背景

布隆过滤器结合了 bit map 和 hash table 的优点，前者节约内存，后者可以处理除了整型以外的很多问题。而布隆过滤器时一种基于二进制向量和一些列随机函数的数据结构，它也有缺点，即有一定的 False Positive 的概率并且不支持删除操作。

布隆过滤器用于判断一个元素是否在集合中，元素越多 False Positive Rate 越大，False Negative是不存在的。

## 2 过程

一个空的布隆过滤器是一个有m bits的bit array，每一个bit都初始化0，并且有k个不同的hash function（m为过滤器的slot数量，k为过滤器重hash function数量）。

![img_0a07faaecf8749d2596c4ec23742623b](/img/in-post/布隆过滤器.assets/img_0a07faaecf8749d2596c4ec23742623b.jpg)

当add一个元素，用k个hash function将它hash对应到过滤器中k个bit上，将k个bit设置为1。

当query一个元素，用k个hash function将它hash得到k个bit位，查询这k个bit，若都为1则返回true（可能有误判）；但是其中有不是1的bit，那么一定不在集合中，返回false。

## 3 优势

当认为可以承受一些误报时，布隆过滤器比其它的集合数据结构有巨大的空间优势。但是元素范围不是很大，并且大都在集合中，则使用确定的bit array远远胜过布隆过滤器。因为bit array对于每个可能的元素空间上只需要1bit，add和query的时间复杂度只有O(1)。当然也是一个k=1的特殊布隆过滤器。

当考虑到冲突的时候，如果要保证只有1%的误判率，则这个bit array只能存储m/100个元素，因而有大量的空间被浪费，同时空间复杂度急剧上升，这显然不是空间效率的。解决的办法很简单，使用k>1的布隆过滤器，即k个hash function将每个元素改为对应k个bit，误判率会降低很多。

## 4 分析

布隆过滤器里的Hash Function需要彼此独立并均匀分布。而且速度上也要尽可能地快，因此类似sha-1就不是一个很好地选择。

布隆过滤器的一个优良特性就是可以定义误判率，当然代价就是更大的过滤器容量。根据误判率推导公式只要先确定可能的容量大小n，然后再调整k和m就能配置带有期望的错误率过滤器。关于选取k和m的高级部分，请自行搜索相关的论文资料。