+++
title="HyperLogLog"
description="用于低开销估算多集合（multiset）的基数（count-distinct）的算法"
date=2022-02-27T17:15:52+08:00

[taxonomies]
categories = ["ProbabilisticDataStructure"]
tags = ["data structure"]

[extra]
toc = true
comments = true
+++

基数统计问题（*Count-distinct problem*）是指在具有重复元素的数据流中找到不同元素的数目的，该问题有许多场景，比如统计通过某个路由器的ip数量、统计访问某网站的用户数以及数据库总某一字段的基数。

可以先看这个视频对本文有直观的认识

{{ youtube(id="jD2d7jr7z1Q") }}

## B-tree

首先，基数统计的难点有：

1. 查找元素

2. 插入元素

由于海量数据存储于磁盘，我们自然会考虑到B树。这里不细展开，只说优缺点。虽然B树查找、插入效率与内存占用上都很均衡，但是每一条记录实际都被完整存储，对于基数统计而言是没必要的；并且B树之间无法**合并**，对于网站场景，想统计两个页面访问用户数时，除非特意建立两个页面的B树，否则无法办到。

## HashSet/Bitmap

为了克服B树遇到的问题，我们自然想到用HashSet减少存储的key值，更进一步将元数据内容转为数字作为index，将key也省略掉，也就是使用Bitmap来统计。既可以快速查找、插入元素，也可以合并。

但是Bitmap有着天然的缺陷，它的长度预先设置，取决于数据的范围（也就是值域，domain）。假设我有10GB个数需要统计，那就需要分配10GB的Bitmap，就算这10GB的数全都是一个。

> 这里补充一句，可以使用压缩位图来**缓解**这个问题，比如roaring-bitmap。

## Linear Counting(LC)[^1]

到这里我们可以发现，想要查找快、插入快、可以合并且内存占用低的数据结构恐怕人类还没掌握。不妨思考人类的常用手段：近似，我们可以试着利用概率去估计出基数。

需要：

* q个输入元素$\overrightarrow{a}=\{ a_0, a_1, a_2, .., a_{q-1}\}$

* 长度为m的Bitmap

* 一个结果服从均匀分布的hash函数，值域[0, m-1]

输出

* 元素集合$\overrightarrow{a}$的基数估计$n$

特点

* 空间复杂度相对于Bitmap仅仅有常数级别降低，因此仍然不适用于大数据集场景

### 基本思想

![LC图示](https://raw.githubusercontent.com/oliverdding/imaw.io/main/linear_counting.drawio.svg)

输入元素集合$\overrightarrow{a}$服从均匀分布，其hash结果$hash(\overrightarrow{a})=\{b_0,b_1,…,b_{q-1}\},0\le{}b_i \le{m-1}也$服从均匀分布。将Bitmap中下标$b_i, 0\le{}i\le{}q$位置1，统计Bitmap中仍为0的个数记为$u$，则$n$的最大释然估计$\hat{n}=-mlog(\dfrac{u}{m})$

> 数学证明非常庞大复杂，请读者自行阅读论文，我就不班门弄斧了。

## LogLog Counting(LLC)[^2]

LC算法空间利用率仍然不高，是否可以进一步优化呢？这就是LLC。

让我借用论文的例子，对于伯努利过程A：不停抛掷一枚均匀硬币，出现反面则继续抛掷，出现正面则停止抛掷，抛掷出的反面次数为$k$。可以得知$P_A(x=k)=(\frac{1}{2})^{k}\quad{}k=0,1,2,3,4,5,…$

思考两个问题：

1. 进行$n$次过程A，每一次过程的总抛掷次数都不大于某个常数$k_{max}$的概率为？

   易得$P_1(x\le{}k_{max})=[1-(\frac{1}{2})^{k_{max}}]^n$

2. 进行n次过程A，至少有一次过程的总抛掷次数等于某个常数$k_{max}$的概率为？

   易得$P_2(x=k_{max})=1 – P_1(x\le{}k_{max}–1)=1-[1-(\frac{1}{2})^{k_{max}-1}]^n$

由于

* $n >> 2^{k_{max}}, \quad P_1(x \leq k_{max}) \approx 0$

* $n << 2^{k_{max}}, \quad P_2(x = k_{max}) \approx 0$

因此我们可以认为n和$2^{k_{max}}$相近～

需要：

- 输入元素集合

- 分桶位数$l$，得到$m=2^l$个桶

- 一个结果服从均匀分布的hash函数，结果长度为$L$

输出

- 元素集合的基数估计$n$

特点

- 标准差为$\dfrac{1.3}{\sqrt{2^l}}$
- 空间复杂度$O(log(log(n)))$

### 基本思想

说了这么多，那么到底有什么用呢？接下来我就从易于理解的方式介绍LLC。

在hash的过程，若hash结果固定，而且hash函数随机性很好，hash函数不尝试碰撞，那么可以认为每一个元素进行哈希后的值转化为2进制串的结果是一个伯努利过程。因此$k_{max}$为从二进制字串高位开始第一个1的位置，$2^{k_{max}}$为基数n的一个估计。

但是单次实验偶然性造成的误差会很大，要如何避免呢？增加几个hash函数，每次插入时并行计算？LLC采用了分桶计算的方式。将hash串前$l$位作为bucket编号，后面的位继续做LLC估算（由于伯努利过程每次实验相互独立，因此去掉前面$l$位并不影响后续的伯努利过程）。假设分了m个桶，第$i, 0\le{}i\le{}m-1$个bucket的第一个1的位置最大值为$M_i$，则LLC的结果为$\hat{n}=2^{\frac{\sum_{i=0}^{m-1}(M_i)}{m}}$。

![分桶重复实验](https://raw.githubusercontent.com/oliverdding/imaw.io/main/loglog_count.drawio.svg)

> 本节介绍的LLC并不是无偏估计，更逻辑严密的推导过程详见论文。

## HyperLogLog(HLL)[^3]

HLL是LLC的改进，用**调和平均数**取代了**算术平均数**。

算术平均数对于离群值十分敏感，在基数总数比较小时，可能会存在比较多的空桶，会干扰平均数的稳定性。

HLL的结果不加证明给出$\hat{n}=\dfrac{\alpha_m m^2}{\sum 2^{-M}}$，其中$\alpha_m$为后续无偏渐进化的参数$\alpha_m=(m\int _0^\infty (log_2(\frac{2+u}{1+u}))^m du)^{-1}$

特点

- 标准差为$\dfrac{1.04}{\sqrt{m}}$
- 空间复杂度O(log(log(n)))

HLL的代码实践可以参考[stream-lib/HyperLogLog.java at master · addthis/stream-lib (github.com)](https://github.com/addthis/stream-lib/blob/master/src/main/java/com/clearspring/analytics/stream/cardinality/HyperLogLog.java)

## 参考

[^1] Whang, Kyu-Young et al. "A linear-time probabilistic counting algorithm for database applications." *ACM Trans. Database Syst.* 15 (1990): 208-229.

[^2] Marianne Durand, Philippe Flajolet. "Loglog Counting of Large Cardinalities (Extended Abstract)"

[^3] Philippe Flajolet and Éric Fusy and Olivier Gandouet and Frédéric Meunier. "HyperLogLog: the analysis of a near-optimal
  cardinality estimation algorithm"
