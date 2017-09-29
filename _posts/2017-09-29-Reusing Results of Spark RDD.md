---
layout: post
title: "Spark RDD重用的一种方法"
comments: true
date: 2017-09-29
description: "RDD重用"
tag: Spark,Paper

--- 


# RDD Share：Reusing Results of Spark RDD

最近在研究一些Spark成本优化的东西，看了一些论文稍微总结一下思路，方便思维拓宽和希望与大家交流！

本篇博文参考自：
>  2016 IEEE First International Conference on Data Science in Cyberspace：
> 《RDD Share：Reusing Results of Spark RDD》

---
## 文章概述及问题描述

Spark作为当下最受关注的分布式计算框架，以其在内存中迭代的特点而逐渐被工业界广泛使用。对于单个用 户，Spark所提供的cache功能（持久化）可以让某些RDD的中间结果在同一个App中的不同jobs间进行共享，但是对于线上的应用，可能同时会用出现多个用户同时发生操作或是前后发生不同操作但是可能会产生相同的中间数据，此时如何管理这些需要需要被重用的RDD使其能够被不同用户所访问是当下Spark不提供的功能。  

** 那么为何要进行这样的工作呢？**
首先是为了提高资源的利用率并降低计算成本，相同RDD缓存后，其他app一旦遇到相同RDD的数据，就可以直接拿过来计算，并不需要重复计算；SQL的常用操作大多都是查询、插入等基本指令，因此重复性的工作是需要避免的，这样可以减轻数据仓库的负债压力。举个小例子：脸书通常情况下会持久化这样的RDD长达天之久。足可见，对于一个大型的分布式系统，这样的考虑和改进是必不可少的。

本片论文就是基于Spark SQL（虽然目前主流的是Spark dataframe以及dataset）提出一种RDDShare系统来管理这些RDD并调度他们在不同用户间进行重用。

---
> 论文中可能会涉及到一些基本的spark的原理等相关内容，在本篇博客中就不详细解释了，如果想要有所了解，[Spark官](http://spark.apache.org/docs/latest/)网是比任何书籍都要通俗易懂的教程，之前自己看源码学习spark的笔记会慢慢整理搬到线上的。请各位大佬多多支持。

---
## 论文牵扯到的Spark基础
1. **Spark SQL**
Spark SQL（下文简称ssql）可以操作类似数据库表或是表单格式的结构化数据，用户提交查询或其他请求给 ssql，ssql会将你的请求代码转化为RDD集合，RDD前后就会产生依赖关系，并最终讲这样的RDD集合提交给Spark Core（sc）来进行计算。sc的高层调度器DAGScheduler会首先将RDD集合转化为DAG图，然后将具体要执行的tasks信息由底层调度器TaskScheduler来分发到各个节点的Executor中去执行。在使用RDD的编程接口的时候，用户可以使用持久化算子来将需要重复使用的RDD进行缓存，以提高作业执行的效率。

---
2. **Catalyst**
Spark SQL的Catalyst优化器将sql转化为可执行的RDD需要经历以下几步（以查询命令为例）：
1> 将查询语句转化为未被解析的逻辑语法树
2> 解析语法书
3> 基于规则来优化语法树
4> 此时的语法数依然是逻辑的，为了能够让Spark读懂，此时再转换为许多的物理语法（即给各个操作绑定上应有的数据结构、表的位置等元数据信息）
5> 基于成本优化模型选择最优执行
6> 将每一个物理语法转换为RDDs
最后RDDs将在Spark Core中进行计算

---
## 方法概述
RDDShare系统能够对多DAG中出现相同task的RDD进行管理，并能够自动在多DAG中去识别相同的RDD，将其重用于其他DAG的计算。

---

## 相关已有研究成果
相关研究分为传统数据库的缓存技术以及云平台下的缓存。
1.  传统数据库
语义缓存： 缓存查询结果及其相应的语义信息。
      \--->表缓存
      \--->动态试图缓存
      \--->高速缓存
      \--->OLAP block cache
      >参考：CAI Jian-yu, YANG Shu-qiang. A Survey of Semantic Caching in Relational Databases. COMPUTER ENGINEERING & SCIENCE. Vol.7, No.10, 2005
 
2. 云平台
    \--->Parag：base on mapreduce，扫描数据文件匹配到用户指令，进行数据共享[1]
    \--->Tomasz：base on mapreduce，合并相同作业，另外还使用了成本模型[2]
    \--->Iman：保存历史中间结果[3]
  >参考：
  >[1] P. Agrawal, D. Kifer, and C. Olston. Scheduling shared scans of large data files. Proc. VLDB Endow.(PVLDB),  1(1):958–969, 2008.
  >[2] T. Nykiel, M. Potamias, C. Mishra, G. Kollios, and N. Koudas. MRShare: sharing across multiple queries in MapReduce. Proc. VLDB Endow. (PVLDB), 3(1-2):494–505, 2010.
  >[3] Iman Elghandour, Ashraf Aboulnaga. ReStore: Reusing Results of MapReduce Jobs. Proc of the 38nd VLDB Conf[C], 2012. 2150-8097.
  
   
## RDDShare System
> * RDDShare System在下文简称 RSS。*

#### 目的
可以对多张DAG进行分析，并自动发现相同的Task，实现跨DAG对RDD缓存数据进行重用。

---
#### 举例
下面主要以两个例子来阐述RSS

Query1：返回年龄大于18的用户信息以及网页信息
```scala
val users = sqlContext.read.json("users.json")
val page_views = sqlContext.read.json(“page_views.json")
val rddjoin= users.filter("age > 18").join(page_views, page_views("user") ===      
users("name"))
val result = rddjoin.show()
```
---
Query2：根据用户名进行分组，并返回超过18岁的用户的总体估计收入
```scala
val users = sqlContext.read.json("users.json")
val page_views = sqlContext.read.json(“page_views.json")
val rddjoin = users.filter("age > 18").join( page_views, page_views("user") === users("name"))
val rddgroup = rddjoin.groupBy(users("name"))
val rddagg=rddgroup.agg(sum(page_views.col("est_revenue" ) ) )
val result = rddagg.show()
```

---
我们都知道，由于Lazy特性，每个RDD并不会立即被执行，而是触发了最后的action的算子才会向前回溯执行算子。执行开始后，最后一个RDD需要等待前面所有依赖的RDD执行完毕后才能开始执行，因此，RDDn的总时间=RDDn的结束时间。可以下面公示表达：
其中M是RDDn的全部依赖集合。
![](http://latex.codecogs.com/gif.latex?%24%24T%28RDD_n%29%20%3DEC%28RDD_n%29&plus;max_%7BM_i%7D%20%5BT%5BRDD_i%5D%5D%24%24)

---
下面我们来对两个例子的DAG图进行分析：

Query1的DAG图如下：
<div align=center><img width="500" height="700" src="http://ieeexplore.ieee.org/mediastore/IEEE/content/media/7864767/7866072/7866153/7866153-fig-3-large.gif" alt="Query1的DAG"/></div>

Query2的DAG图如下：
<div align=center><img width="500" height="700" src="http://ieeexplore.ieee.org/mediastore/IEEE/content/media/7864767/7866072/7866153/7866153-fig-4-large.gif" alt="Query2的DAG"/></div>


---
不管是从图中还是从代码中我们都知道其中**join**算子是被两次重复使用的，并且join的两个表都是一样的，那么如果我们在执行Query1的时候可以持久化join后的结果那么对于Query2的执行时间就会缩短至：![](http://latex.codecogs.com/gif.latex?%24%24%20T%28RDD_n%29%20%3DEC%28RDD_n%29&plus;max_%7BM_r%7D%20%5BT%5BRDD_r%5D%5D%24%24)
其中 ![](http://latex.codecogs.com/gif.latex?%24max_%7BM_r%7D%20%5BT%5BRDD_r%5D%5D%24) 代表的是读取M子集中所用的最长时间。

---
假设目前有个子集N并没有被缓存，并且![](http://latex.codecogs.com/gif.latex?%24N%5Cin%20M%24)，显而易见，这样的持久化效果是没有之前好的。
**因此重点及难点是需要尽可能找到能够被重复利用的RDD**

---
因此文章提出第二种持久化模型： *缓存部分 RDD*
例如Query1中的filter算子假如刚好也出现在另外一个DAG中，那么我们可以持久化这个filter算子的计算结果，并rewrite到Query1中，相当于Query1的代码可以省略filter之前的一些依赖算子，只需要直接加载之前缓存到的数据即可。如果是这样一种情况，那么Query1的DAG就会变成如下：

Query1的rewrite-DAG如下：
<div align=center><img width="500" height="700" src="http://ieeexplore.ieee.org/mediastore/IEEE/content/media/7864767/7866072/7866153/7866153-fig-7-large.gif" alt="Query1的rewrite-DAG"/></div>


其中[42]的MapPartionRDD就是之前被缓存过的中间结果。
此时，该模型可用下面的公式来表达：

![](http://latex.codecogs.com/gif.latex?%24%24EC%28RDD_n%29%20%3D%20T_%7Bload%7D&plus;T_%7Btransformation/action%7D&plus;T_%7Bshuffle%7D&plus;T_%7Bstore%7D%24%24)

其中 ![](http://latex.codecogs.com/gif.latex?%24T_%7Bload%7D%24&plus;T_%7Bshuffle%7D&plus;T_%7Bstore%7D%24%24) 就代表读取已有缓存所耗费的时间，![](http://latex.codecogs.com/gif.latex?%24T_%7Btransformation/action%7D%24) 代表所有Transformation级别算子的耗时，![](http://latex.codecogs.com/gif.latex?%24T_%7Bshuffle%7D%24) 代表宽依赖算子所要消耗的时间。![](http://latex.codecogs.com/gif.latex?%24T_%7Bstore%7D%24) 代表存储结果所要花费的时间，一般情况下，也只有发生cache等持久化或是遇到Action级别算子（典型的saveAstextFile）的时候才会产生存储的时间耗费。

###RDDShare的组件
#####输入输出
> 输入：DAGs
> 输出：rewrite后的DAGs

---
#####核心组件
>DAG Matcher：DAG的匹配器
>RDD Cacher：RDD的缓存器
>DAG Rewriter：DAG的复写器

---
#####工作流程
先给出一副工作流程图，基本上光看图也能明白这三个核心组件是怎样在工作的

<div align=center><img width="500" height="700" src="http://ieeexplore.ieee.org/mediastore/IEEE/content/media/7864767/7866072/7866153/7866153-fig-8-large.gif" alt="RDDShare 的工作流程图"/></div>


根据图中所展示的流程，简要介绍一下RDDShare的工作机制： 
>1. 首先我们需要知道Waiting Queus里面存放的是正在排队的DAGs，也就是多个job在排队，可能每一个job就是一个查询语句或是一段复合代码，这些都有可能，视Action算子的位置而定。
>2. 其次，这个等待队列中有两个重要区域，前n个DAG被存放在matching window，用于提交到Spark Core中去执行，超过n的n+m个DAG被放置于candidate window中。
>3. 首先对于第一个DAG（第一个job）会被直接送去计算，因为这个时候并没有缓存的中间结果给他用，此时matching window中就空缺了一个位置，candidate中的DAG会挨个通过**DAG Matcher**与matching window总的DAG进行匹配，匹配的目的是为了寻找到重复次数最高的RDD先进行缓存。
>4. 寻找这样一个RDDMAX就是用matching中的每一个DAG拿去给candidate中的DAG重复去比，寻找到两两间重复次数最高的那个DAG，一有重复的就记录一个 $Repeat_j$，最后找到重复次数最高的那个 ![](http://latex.codecogs.com/gif.latex?%24Repeat_j%24)，其对应一个DAGMAX以及RDDMAX，与之对应的原有的candidate window 中的DAG可能会有多个，记作 ![](http://latex.codecogs.com/gif.latex?%24DAG_%7Bimax%7D%24) 以及 ![](http://latex.codecogs.com/gif.latex?%24%7BRDD_%7Bimax%7D%7D%24)
> 5.此时RDD Cacher会持久化这个RDDMAX到内存或者磁盘，随后，立马通过DAG Rewriter将持久化后的RDD直接复写到 DAG_imax中，并依次将 DAG_imax 压入matching window中。

---
至此，整个系统的工作原理介绍完毕。

###总结与延伸

这篇文章篇幅较短，基本上精度一遍也花不了太多时间。虽然这样的研究是基于Spark-1.5，但是对于目前的版本仍能够提供很好的思路。后续自己将做一下延伸，个人感觉本片论文的对于处理结构化数据时候的复用思路是值得推广并有很大使用价值和效益意义。

如果对这方面有感兴趣的童鞋欢迎评论或私信讨论哈。

---

**我的简书 : <http://www.jianshu.com/p/0090ca6a1bd9> **

**转载请注明原址，谢谢*。
