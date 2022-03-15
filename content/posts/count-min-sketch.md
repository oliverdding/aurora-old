+++
title="Count-Min Sketch"
description="不精确但低消耗地统计海量数据流中每个元素的个数"
date=2022-03-15T22:58:53+08:00

[taxonomies]
categories = ["ProbabilisticDataStructure"]
tags = ["data structure"]

[extra]
toc = true
comments = true
+++

本文将介绍[概率性数据结构](/categories/probabilisticdatastructure/)这个专题的第三个数据结构，实际上它只是[布隆过滤器](/posts/bloom-filter-1/)的一个变体，它的结构几乎和计数布隆过滤器一致，但是解决不同的问题。

在上篇文章[HyperLogLog](/posts/hyper-log-log/)中我们提到基数统计问题，换个问题，如果要统计海量的数据流中每个元素的出现次数呢？

一看这个问题我们自然又会想到hash数据结构，但相比于[HyperLogLog](/posts/hyper-log-log/#hashset-bitmap)一文中使用的HashSet，我们需要使用HashMap用value作为counter计数。

理所当然，原型方案只是用于提供一个解决问题的思路与方式，我们需要考虑它的局限。假设流中有千万个数据，若它们都不一样，那HashMap中就有千万个键值对，空间占用可想而知。本文介绍的Count-Min Sketch(CMS)就是利用概率的方式解决空间消耗过大问题。

:warning: 相比于论文[^1]，本文的查询只涉及point query，而不涉及range query和inner product query。

可以先看这个视频对本文有直观的认识

{{ youtube(id="mPxslXpg8wA") }}

## 利用Hash节省空间

首先想到的方案是利用Hash函数和数组，将元素Hash结果作为index，在数组中统计，这样就可以节省掉HashMap中存储key的空间。

首先准备一个Hash函数$h$，一个长度为$w$的数组$count$。

* 初始化时将$count$数组元素初始化为0：$count[i] = 0, 0\le{}i\lt{}w$

* 添加元素时，对元素$a$计算Hash值并对$w$取模，作为下标将$count$数组中对应元素自增：$count[h(a)\mod{}w]+=1$。

  ![插入元素示例](https://raw.githubusercontent.com/oliverdding/imaw.io/main/single-array-in-count-min-sketch.drawio.svg)

* 查询元素时，同样方式获取下标然后返回数组元素内容：$count[h(a)\mod{}w]$。

这种方式有一个致命缺陷：Hash碰撞。一旦发生Hash碰撞，意味着碰撞的两个元素的统计值（两个元素的个数总和）会大于实际值，也就是说返回的值一定是上限，大于等于真实值。想缓解这个问题，看起来我们只能分配一个足够大的数组，并且运用一些Hash碰撞的解决办法。那么有没有更简单的办法呢？

## CMS

既然一个Hash函数+数组容易发生碰撞，那么如果引入多个Hash函数与多个数组呢？这就是CMS的解决办法。

首先准备$d$个Hash函数$h_1,h_2,...,h_d$，一个宽度为$w$深度为$d$的二维数组$count$。

![空count数组](https://raw.githubusercontent.com/oliverdding/imaw.io/main/empty-cms.drawio.svg)

* 初始化时将$count$数组元素初始化为0：$count[i, j] = 0, 0\le{}i\lt{}w, 0\le{}j\lt{}d$

* 添加元素时，对元素$a$计算$d$个Hash值，并将二维数组中Hash函数对应的一维数组的元素自增：$count[i, h_i(a)\mod{}w]+=1$（$0\le{}i\lt{}d$的全部元素）

  ![插入元素示例](https://raw.githubusercontent.com/oliverdding/imaw.io/main/cms-insert.drawio.svg)

* 查询元素时，同样获得$d$个Hash值，并取得对应的$d$个元素中的最小值：$min_{i=0}^{d-1}count[i, h_i(a)\mod{}w]$（$0\le{}i\lt{}d$的全部元素）

  ![查询元素示例](https://raw.githubusercontent.com/oliverdding/imaw.io/main/cms-query.drawio.svg)

通过多个Hash函数与多个数组（一维数组）的方式，尽可能避免Hash碰撞的影响，这就是CMS的全部内容，一般将底层的二维数组称为sketch。

可是如何能证明它的的概率可用？$d$、$w$应该取什么值呢？

## 理论部分

论文引入了两个符号$\varepsilon$和$\delta$，这两个都是用户根据自己可用资源和所需目标给定的值，除此之外令$\parallel{}count\parallel{}_1$为sketch所有元素的总和。

**理论**： 用户设定$\varepsilon$和$\delta$后，可以根据公式$$w=\lceil{}\frac{e}{\varepsilon}\rceil{}$$和$$d=\lceil{}ln\frac{1}{\delta}\rceil{}$$计算$w$和$d$的值。论文[^1]证明，可以在$1-\delta$的概率内，误差最大为$\varepsilon{}\ast{}\parallel{}count\parallel{}_1$

主观上体会，增加新的Hash函数可以降低超出最大误差的概率，因为新增后还是超出最大误差需要新增的Hash函数也发生Hash碰撞，而不同Hash函数间相互独立，具有指数效应。

增加宽度有助于分散计数，降低最大误差，但是这只是线性的。

一般而言，sketch的宽度（$w$）会远远大于深度（$d$），过多的Hash函数（$d$）会造成额外的开销。同时，随着Hash函数（$d$）增多，$\delta$会极速缩小。

## 参考

[^1]: Graham Cormode and S. Muthukrishnan. 2005. An improved data stream summary: the count-min sketch and its applications. Journal of Algorithms 55, 1 (2005), 58–75
