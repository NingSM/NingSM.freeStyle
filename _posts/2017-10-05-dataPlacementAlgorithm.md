---
layout: post
title: "用于数据负载平衡的spark中间数据存储策略"
comments: true
date: 2017-10-8
description: "shuffle数据倾斜问题的分析与解决"
tag: Spark

---

# An intermediate data placement algorithm for load balancing in Spark computing environment
最近在研究一些Spark成本优化的东西，看了一些论文稍微总结一下思路，方便思维拓宽和希望与大家交流！

本篇博文参考自：
>  Future Generation Computer Systems 78 (2018) 287–301：
> 《An intermediate data placement algorithm for load balancing in Spark computing environment》

---
## 1  文章概述及问题描述

Spark作为基于内存迭代的云计算框架，其很容易发生数据倾斜，尤其是在Shuffle阶段，reduce端所拉取的数据量很容易出现不平衡，这将导致某些reduce计算很久，使得整体计算发生延时，严重时会导致application失败。本篇论文讨论spark中mapreduce框架中所出现的shuffle阶段的数据倾斜问题。

MR的数据格式全是K-V形式，因此数据的倾斜就是Key的倾斜，导致某些分区中的数据量过大，因此，对于reducer来说，partition的倾斜将会导致reduce的倾斜。

* 为什么容易出现数据倾斜 *
partition的大小取决于具有相关性的key/value的数量，由于spark中keys的分配调取由hash算法决定，因此不同的reducer之间会出现数据量不同的情况，有些reducer要处理的数据可能会非常大。


---

## 2  论文中的相关术语以及spark基础
1. **clusters**
一个data clusters就是一组包含全部相同Key的map中间输出结果的数据集合。
在Spark中，所有的被相同的reduce处理的clustes组成一个分区（partition）。

---
2. ** Bucket **
bucket是指一个序列的缓冲区，用于收集并缓存map的输出，相对应的reduce会从相应的bucket上拉取数据。

---
3. ** M*R **
假设reduce个数为R，map个数为M，则每一个map将会创建R个bucket，所以bucket的总数为R*M

---
4. ** spark-shuffle（1.1.0）中的输出分布 **
![Data distribution of shuffle in Spark 1.1.0.](https://ars.els-cdn.com/content/thumbimage/1-s2.0-S0167739X16302126-gr1.sml)

---
5. ** Zipf distributions **
文本数据的key一般来说是服从Zipf distributions
> 参考：J. Lin, et al. The curse of zipf and limits to parallelization: A look at the stragglers problem in mapreduce, in: 7th Workshop on Large-Scale Distributed Systems for Information Retrieval, 2012, pp. 2000–2009.

---

## 3  方法概述
为了解决shuffle阶段出现的数据倾斜，本文提出了一种分割与合并的算法（splitting and combination algorithm for skew intermediate data blocks (SCID)），这种算法采用基于水塘抽样的抽样算法，来自动侦测中间数据的分布情况，并且是在spark程序运行之前就可以判断。

大致做法就是SCID会对键值集合的clustes的大小进行排序，并且将会依次存放在相应的bucket中，bucket的大小是固定的，某个bucket存满后，该clustes会被切分，剩下的clusters将进入下一次的迭代。经过这样的几轮分配，每一个bucket中的数据大小几乎相同，所以此时相应的reducer拉取到的数据就是相同的。与此同时，我们可以知道，同一个bucket中可能会存放来自不同clusters的数据。

但是，上述算法所遇到的问题就是，如果map中间输出数据的key的分布实现不知道的话，就无法在clusters上使用精确的判别机制来进行切分。而这样的算法只有在MR执行前进行才会显得有意义，因此本文提出动态范围分区（dynamic range partition）的方法，对输入数据进行以此预先的抽样，并将抽样结果输入给一小部分的map tasks中，以** 实现分布统计 **，通过相应的统计值，可以预测出map阶段后所产生册所有clusters的大小，这个结果将会作为split策略的输入。

* 注意，所有的策略均在真正spark程序运行之前进行 *

综上所述，本文所做的工作就是在原有spark架构基础上添加了一个统计分布功能用于抽样并作为split和combine算法的输入。这是一种细粒度（fine-grained）的算法，他能够重新进行partition，改变原有分区的大小并发送给相应的buckets，从而解决了reduce端数据倾斜问题。

---

## 4  主要贡献
> 使用水塘抽样进行输入数据的采样，并提出一种验证模型来选择合适的采样率。** 这样的模型在运行中考虑到了成本、效果以及方差的重要性 **。
> 提出了一种切分以及合并键值对clusters的算法。通过以相同大小的clusters的组合来填充固定数量的buckets从而达到reduce端具有更好的负载均衡。
> 文章基于spark1.1.0验证了本文算法模型带来的性能提升和数据倾斜问题的减缓。

---
   
## 5  系统整体介绍
> 由前面的术语介绍以及mapreduce的原理可知：每一个split会由一个map来处理，并生成一中间结果，中间结果通常是被partitioned的，而且被分到相应的bucket中去。本文使用如下表达式来来自 m map Tasks的中间结果：
> ![](http://latex.codecogs.com/gif.latex?I%5Csubseteq%20K%5Cast%20V)
> 根据bucket的概念，我们可以用下面的公式来表达一个cluster所分成的buckets：
> ![](http://latex.codecogs.com/gif.latex?C_k%20%3D%20%5Cleft%20%5C%7B%20%28k%2Cv%29%5Cin%20I%20%5Cright%20%5C%7D%2Ck%5Cin%20K%2Cv%5Cin%20V)
> 所有属于相同cluster的键值集合被放置在同一个partition中，所以根据key可以将所有的K进行n个partition的划分，用如下表示：
> ![](http://latex.codecogs.com/gif.latex?%5CPi%20%3A%20K%5Crightarrow%20%281%2C%5Ccdots%20%2Cp%29)

下图流程是一个被改善过的Spark工作流程。
![]()

其中最重的组件就是 ** load balancing module **。spark运行之前，这个组件会生成一个balanced partitioning策略，这个策略会指导该如何split clusters以及如何combine分割后的clusters。balance策略能够被使用在shuffle阶段。
关于** load balancing module **，在本文共包含以下两个阶段：
> 1. Data samling：抽样是在map之前进行的，所谓的抽样就是从全部的输入split采样小样本，进而通过对key的统计方法来预测map输出结果中每一个cluster的大小。
> 2. Splitting and combination：抽样结束后，会产生一个clusters的切分和combine方法。一些过大的clusters可能会被切分多个以填充到不同的buckets中去。切分的主要还是要看固定的bucket的容量大小。

---

## 6  抽样模型
#### 6.1 数据偏移模型
> 模型的本质就是对clusters、bucket等概念进行结构化，通过对每一个组件进行结构化的分析，最终引出偏移与均值和方差的条件关系，推导出FoS这样一个偏移程度的指标。因此，文中定义以下几个公式：

1）因为一个cluster中是包含相同key的键值对数据，因此，所有的clusters能够被形式化如下：
` C=｛C1,C2....Ci...,Cm｝,  1<=i<=m `
m是cluster的编号，Ci被定义为一种结构体：` Ci =｛order,SC｝ `，其中order记录了这个cluster的顺序，而SC被定义为一个序列：` SC = {SC1,SC2....,SCi...SCm}  1<=i<=m `.其中SCi是整形变量，表示对应的cluster的数据集的大小。
2）bucket可以形式化为：` B = {B1,B2,....Bk,...Bn}  1<=k<=n ` 其中n为bucket的编号 

3）前面说到，文本数据的key一般服从Zipf分布，这个模型里面设置了一个变量a（0.1<=a<=1.2）来控制偏移量，a越大代表偏移越大，a仅仅影响当前输入数据的偏移大小。本文使用矩阵P来表示这样一种分布，pki表示从cluster Ci所获得的键值对的数量，这个数量后面将会被bucket Bk所拉取（说到这，大家还是要搞清楚文中这几个术语之间的关系的：map的输出数据相当于一堆键值对，拥有相同key的数据<tuples>组成一个cluster，而同时被相同reduce处理的clusters组成一个partition，partition最后会被放入bucket中），因此SCi可以被看作是在所有buckets中来自cluster Ci的总大小，可以用如下公式来定义：
![](http://latex.codecogs.com/gif.latex?SC_i%20%3D%20%5Csum_%7Bk%3D1%7D%5E%7Bn%7D%20p_%7Bk%2Ci.%7D)

4）bucket k中所包含的键值对的个数由` BC(k) ` 来表示，对于数据的分布pki，外加上之前给的变量a，则分布可以表示为：
![](http://latex.codecogs.com/gif.latex?p_%7Bk%2Ci.%7D%5Ea)
那么BC(k)的值可以表示为如下公式：
![](http://latex.codecogs.com/gif.latex?BC%28k%2Ca%29%20%3D%20%5Csum_%7Bi%3D1%7D%5E%7Bm%7Dp_%7Bk%2Ci.%7D%5Ea)
5）由上式，我们可以计算在当前偏移参数a的前提下的所有buckets的键值对的平均数量如下公式所示，其中，n代表的是全部bucket的数量。
![](http://latex.codecogs.com/gif.latex?mean_a%20%3D%28%20%5Csum_%7Bk%3D1%7D%5E%7Bn%7DBC%28k%2Ca%29%20%29/%20n%20%3D%28%20%5Csum_%7Bk%3D1%7D%5E%7Bn%7D%20%5Csum_%7Bi%3D1%7D%5E%7Bm%7Dp_%7Bk%2Ci%7D%5Ea%20%29%20/%20n)
6）一般而言，被bucket所处理的中间数据能够被看作是具有数据偏移标准差的，那么，当满足以下条件时，被bucket所处理的clusters将被视作具有数据偏移,std是所有buckets中key-value对个数的标准差：
![](http://latex.codecogs.com/gif.latex?%7Cmean_a-BC%28k%2Ca%29%7C%3Estd)
7）所有clusters中的数据的标准差可以用以下公示表达：
![](http://latex.codecogs.com/gif.latex?std%28P%2Ca%29%3D%5Csqrt%7B%5Csum_%7Bk%3D1%7D%5E%7Bn%7D%20%28%5Cleft%20%28%20%5Csum_%7Bv%3D1%7D%5E%7Bn%7D%20%5Csum_%7Bi%3D1%7D%5E%7Bi%7D%20p_%7Bv%2Ci%7D%5Ea%20%5Cright%20%29/n-%5Csum_%7Bj%3D1%7D%5E%7Bm%7Dp_%7Bk%2Cj%7D%5E%7Ba%7D%29%5E2/n%7D)
8）最后，为了归一化表达数据偏移的程度，定义了FoS指标（factor of skew）来评测所有buckets负载均衡的程度,FoS指标值越小，则代表负载越均衡，则偏移也相应的越小：
![](http://latex.codecogs.com/gif.latex?FoS%3Dstd%28P%2Ca%29/mean_a)

#### 6.2 水塘抽样算法
** 为何要使用水塘抽样？ **
在一般的编程语言中，常规的抽样是使用伪随机数，对于大规模的数据，特别是随着采样空间的增加，这样简单的伪随机数不能保证所有样本完全随机化，不可避免地会产生一些重复样本。而水塘抽样则能够有效避免这一问题，他将使得样本出现的概率均相同，保证样本的随机性，特别是从某些序列流数据中抽取数据时，水塘抽样可以保证原始key的分布更加接近于整体真实情况。
水塘抽样是为了解决未知大小数据集的随机数抽取问题，要求从一个未知大小的数据集中等概率地拿出k个数。尤其是在大数据背景下的采样问题，对于大规模数据，我们无法将其全部加载到内存，此时需要根据内存大小k来等概率地从全部数据中抽取大小为k的数。
>水塘抽样基本思想：
> 1、简单场景：如果我们已知这个数据集只有3个数字，那么我们在拿取第一个数的时候，其出现在水池中的概率为1，拿取第二个数的时候，其出现在水池中的概率为1/2，在拿取最后一个数的时候，我们为了等概率地返回1个数，分为两种情况：1）返回第三个数：显然如果要保留第三个数则齐概率为1/3。2）如果返回前两个数的其中一个，则其概率为（1-1/3）*1/2=1/3，即不选择第三个数的概率*选择前两个数任意一个的概率，因此水塘中每个数字出现的概率均相同。
> 2、复杂场景：文中的抽样方法是需要返回k个数，因此这里直接修改1中的返回一个数为k个数即可，即：每个数字在水池中出现的概率为k/n。
> 其算法流程如下：
> 1）初始化时，我们依次将前k个数加载到水池中。
> 2）随后考虑第k+1个数的生死。此时分两种情况：
> a.水池中全部元素没有被替换
> b.水池中某个元素被第k+1个替换
> 先来看情况b：对于第k个元素，此时生成一个0~i（i=k+1）的随机数p，如果p<k（相当于生成0~1的随机数），则第k+1个数被选中，并且用这个数去替换水池中的某一个数，此时第k+1个数在水库中出现的概率为k/k+1，接下来我们要看看水库中每个元素被替换的概率：条件概率，首先要第k+1个数被选中，其次是k个数中随机选出一个来替换，则k个数中被替换的概率为(k/k+1)*(1/k)=1/k+1.那么水库中原先的k个数每个数还能继续出现的概率就等于1-1/(k+1)=k/k+1，可以看出，不管是新来的元素还是以前的元素的出现概率均为k/k+1。 
> 对于情况a：如果所有元素都没有被替换，就说明第k+1个元素没有被选中,则此时水池中每个元素出现的概率就为1-第k+1个元素被选中的概率=1-k/(k+1)=1/k+1
> 这样的一个规律可以用数学归纳法，直到证明完第i+1个数时仍然成立即可。
下面是水塘抽样算法的伪代码部分：
>```
> //stream代表数据流  
> //reservoir代表返回长度为k的池塘  
> //从stream中取前k个放入reservoir；  
> for ( int i = 1; i < k; i++)  
>    reservoir[i] = stream[i];  
> for (i = k; stream != null; i++) {  
>    p = random(0, i);  
>    if (p < k) reservoir[p] = stream[i];  
>return reservoir;
>```
#### 6.3 cluster容量大小的预测
建立在抽样算法的输出结果上，同时，基于抽样算法中的假设：抽样得到的数据的key的分布与整体数据的key的分布是相同的，因此，能够大致预测出每个map节点上每个cluster的数据大小。
步骤：
1. 输入数据是抽样算法的输出结果，用BS表示；同时MRjob用来表示处理BS的spark作业。
2. 基于部署在map节点上的监控工具，可以得到clusters个数的记录，同时获得每个cluster中k/v的个数{SCi}.
3. 由于抽样数据中的每一个key的数量被认为是与原始数据中key的数量成比例的，因此可以通过扩大每一个clusters的个数来来近似估计cluster的真实大小。

## 7 Cluster的切分和重组模型（5. Cluster splitting and combining）
1. 算法流程概述：
clusters的切分和重组算法实际就是想法设法将全部的clusters打包并分配到大小相近的buckets中去。整体流程可参考下图：
![](http://www.sciencedirect.com/science/article/pii/S0167739X16302126#gr5)
算法的输入是前面我们所定义的{Ci}:clusters tuples的集合；{SCi}:每一个cluster在{Ci}中集合的个数；B:当前buckets的序列；RB:当前bucket剩余容量。算法的输出是矩阵P，代表着clusters的存放策略。
整体思想是先对map的中间结果产生的clusters进行降序排序，然后从最大的cluster（SCm）开始判断其与固定bucket（RB）的大小关系，如果SCm>=RB,则对SCm进行切分，只将满足bucke大小的那一部分放进第一个bucket中，剩余的一部分作为一个segment将进入第二轮判断（注意，被spilt的剩余部分在第二轮迭代前会与其他的clusters进行重新排序），以此类推；如果SCm<=RB，则直接将其放入该bucket中然后判断下一个SCm-1是否满足剩余空间大小，以此类推。整个流程分轮次进行迭代判断，也就是说，每一次只会填充一个bucket。其中RB是表示当前bucket剩余空间大小，随着clusters的填充，RB的值会不断更新，某一轮的停止条件为寻找到一个SC的大小大于RB，则表示当前轮的bucket即将被填满。

> 那么问题来了：
> 在spark-1.1.0中，当map的中间输出达到20%时，就会有shuffle阶段启动，那么说明我们是无法等到全部clusters都生成后才去得到切分和重组策略！！！
> 对于这个问题，其实我们需要知道，切分和重组算法其实相当于一个模拟的过程，我们在作业执行前通过采样得到了一组采样数据，这组采样数据能够真实反映整体数据在map输出时的大致分布情况，因此，在进行切分和重组算法时，模型只需输出在已知clusters分布的情况下的一个模拟bucket分配方式即可。

切分和重组算法的输出矩阵P如下图所示：Pij的定义回顾一下：第i个bucket从第j个cluster所拉取的k/v的数量。这个矩阵可以换一种形式来表示，下面的形式将直接表示出每个cluster中被每个bucket所处理的k/v的数量。

2.真实作业时的Cluster分配算法
真实作业时，map达到20%输出时就会进行shuffle，此算法就是根据先前的模拟放置策略来动态决定当前输出数据的存放。算法以矩阵P以及｛Ci｝（每一个clusters的大小）为输入，以｛CBij｝来表示每一个bucket的当前负载情况，即代表当前第i个bucket所拉取的第j个cluster的数据大小。
算法流程：
该算法用于真实作业情况下中间数据的放置。有两种基本情况
1）首先对C进行遍历获得Ki的位置，如果此时在C中找不到Ki，则按照默认的hash来放置Ki。
2）如果找得到，则代表可以在矩阵P中获取到该Ki。算法中实现这一过程其实是去遍历相关列向量pj，在矩阵P中，我们知道列序号代表着key在clusters集合C中的位置，因此，对于向量pj，每一个元素都表示着该key在每个bucket中所被允许的最大的容量。解释完pj，则这个流程可以概括为：在遍历pj的同时，来判断当前每一个bucket的负载情况BCij是否小于pij的极限，也就是为了判断Ki的位置，算法将会遍历所有的bucket。



---

** 我的简书: <http://www.jianshu.com/p/0090ca6a1bd9> **

* 转载请注明原址，谢谢 *。