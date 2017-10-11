---
layout: post
title: "智能RDD管理--用于在Spark中实现高性能内存计算---论文总结"
comments: true
date: 2017-10-05
description: "RDD重用"
tag: Spark-paper

---

# Intelligent RDD Management for High Performance In-Memory Computing in Spark
最近在研究一些Spark成本优化的东西，看了一些论文稍微总结一下思路，方便思维拓宽和希望与大家交流！

本篇博文参考自：
>  [WWW '17 Companion](http://www.www2017.com.au/) Proceedings of the 26th International Conference on World Wide Web Companion：
> 《Intelligent RDD Management for High Performance In-Memory Computing in Spark》

---
## 1  文章概述及问题描述

Spark是当下科研界或工业界中普遍使用的内存计算框架，可以通过将访问的数据包装成为弹性分布数据集（RDD），并将这些数据集存储在快速访问的主内存中，以大大加快计算速度。

但是，毕竟我们的计算内存是有限的，而Spark本身不会提供RDD的合理存储机制，除此之外，如果中间的RDD因为一些原因缺失，则会导致该RDD的所在全部计算链条重新计算，这将带来额外的计算和时间成本。

虽然spark内部提供的粗粒度（coarse-grained）checkpoint接口可以让我们对想要保存的RDD数据进行持久化，但是对于日常的应用业务，其逻辑和代码链条较为复杂，而人们都是靠经验去checkpoint一些RDD，很多时候，我们牺牲了磁盘来保证程序的容错性，这种方式都会导致spark执行效率的降低，磁盘IO以及我们分布式存储系统通常也有一定负担。

此外，spark中还存在一个内存回收的问题，如果内存不够用了，spark内部的LRU[2]算法（least recently used）来清楚一些使用频率最小的RDD，但是该算法只考虑了使用频率，并没有考虑该数据分区的关于成本问题的一些因素，例如其计算成本问题，加入后面需要用到它，到底是计算他的成本小还是一直存储到下一次使用的成本小。

因此，本文就是为了解决RDD的合理存储问题，提出** 一个细粒度（Fine-Grained）的RDD-checkpoint和kick-out选择策略，通过该策略，Spark可以智能地选择合理的RDD进行存储，以最大限度地提高内存的使用率 **。

> LRU参考：
> [2] D. Lee, J. Choi, J. H. Kim, S. H. Noh, S. L. Min,Y. Cho, and C. S. Kim. LRFU: A spectrum of policies that subsumes the least recently used and least frequently used policies. IEEE Transactions on Computers, 50(12):1352-1361, 2001.

---

##  2  论文牵扯到的Spark基础
**1）. RDD**
Spark是用于大数据处理的常用内存计算引擎。 Spark中的关键组件是弹性分布式数据集（RDD），它是一种分布式内存抽象，利用RDD进行编程，我们可以以容错方式在大型集群上提交并执行内存中的计算。 RDD利用内存来缓存中间结果，与其他大型数据密集型框架相比，Spark具有巨大的优势，例如。Hadoop的[1]。

---
** 2）. CheckPoint **
checkpoint的目的是用来将某RDD的数据以及元数据信息进行缓存，内存和磁盘均是其缓存的介质对象。一旦RDD被checkpoint，其前面全部的父节点会被祛除，仅保留checkpointRDD作为其父RDD，这样的机制可以减少RDD的链条长度，便于RDD的恢复。同时可以方便用户跨多个计算进行RDD数据的访问。** 但是这样的粗粒度checkpoint会带来一个严重问题，被切掉的依赖链条中的RDD可能在将来会被用到，因此此时spark必须要知道这些RDD的计算信息，以用于恢复**

---
## 3  方法概述
本文提出了一个智能的RDD管理方案，以帮助解决长链条RDD问题，同时对性能影响较小，并且该方案可以解决内存空间首先的问题。 
首先提出了细粒度检查技术，以避免丢弃高频被使用的的RDD。 这种细粒度的Chekcpoint方法不会立即抛弃该RDD的全部父依赖，而是会去全局判断是否要丢弃。
然后，讨论如何巧妙地checkpoint一些RDD以消除计算开销。
为了解决有限的存储空间问题，提出了一种新颖的kick-out选择方案来对包含宽依赖性的RDDs进行高速缓存。 
文中的实验验证部分是基于Spark-1.5.2，并且使用大量数据工作负载，实验表明该方案比baseline实现了高达28.01％的性能提升。

---
   
## 4  细粒度checkpoint
Lineage可以用于恢复RDD，但是对于长链条的RDD，这种恢复是十分耗时的，因此考虑checkpoint的方法来消除这种开销。

一般来讲，checkpoint对于使用频率高的或是长链条的RDD，基于这样的理论，本文提出如下建模公式来有选择的对RDD进行checkpoint：
![](http://upload-images.jianshu.io/upload_images/8166116-cb5eb702cce4f5da.latex?imageMogr2/auto-orient/strip)

其中L和C分别表示选择重新计算或是checkpoint达到恢复该RDD的所需时间，w代表故障频率，取值范围为[0,1]，如果上式成立，则会进行checkpoint。

---
## 5  Kick-Out 选择策略
LRU策略只考虑了时间维度，但是对于复杂长链条的计算，内存空间的利用率应当是又要考虑的，因为长链条带来的最大的问题就是计算开销，尤其是许多的宽依赖意味着这些RDD将来会有较大概率被访问，所以持久化这些RDD可以在一定程度上提高内存的使用率。

####  5.1  选择策略介绍
主要思想是给具有宽依赖的RDD分配优先级P，优先级决定了使用存储空间的优先级。分配策略如下：

![优先级划分](http://upload-images.jianshu.io/upload_images/8166116-6d18f421da613269.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中L和C就是上面讨论的直接计算和通过checkpoint来计算的时间花费，D代表该RDD的依赖程度（这里文中没有解释太多，我认为依赖程度就是该RDD所依赖的RDD的个数吧），权重就是将其除以最大的值进行进行归一化相加得到，思想还是比较好理解的。

实验验证选取了Stanford和berkStan的graph dataset。实验结果表明最大性能提升28.01%，平均提升13.63%。实验结果如下图：

![result.png](http://upload-images.jianshu.io/upload_images/8166116-594aa2ed124b15b3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

---
至此，整个模型的思路介绍完毕。


**我的简书: <http://www.jianshu.com/p/b4cfac386779>**

*转载请注明原址，谢谢*。
