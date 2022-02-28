+++
title="1 基础布隆过滤器"
description="最普通的布隆过滤器，涉及结构解释和数学推导"
date=2022-02-09T17:13:10+08:00

[taxonomies]
categories = ["ProbabilisticDataStructure", "BloomFilter"]
tags = ["data structure"]

[extra]
toc = true
comments = true
+++

## 介绍

布隆过滤器（Bloom Filter）用于判断某个元素是否在集合中的一个简单、高效、内存占用低的数据结构，它类似于哈希表，相比于后者，它存在false positive值，但显著地降低了内存占用。这种[概率性数据结构](/categories/probabilisticdatastructure/)值得体会，以退为进，*只要false positive控制在可以允许的范围*，资源消耗被大幅降低[4]。

使用布隆过滤器查找key时，返回值可能情况有两种：

1. key**不**存在
2. key**可能**存在（存在、纳伪）

> 使用布隆过滤器原则：false positive影响可控。

### 历史

布隆过滤器于1970年代被Burton Bloom创造，在查询集合成员存在与否时一般考虑时间成本或空间成本，论文提出第三个优化方向：Allowable Fraction of Errors，也就是说允许一定的误判，来大幅降低空间占用[1]。之后一段时间布隆过滤器被广泛运用在数据库领域。

之后又有人提出利用布隆过滤器+共享缓存的方式大幅降低缓存服务器的带宽[2]，自此布隆过滤器在互联网中展现实力[3]。

### 原理

布隆过滤器由以下三部分构成：

1. n个key
2. m位的位图：`bitmap`
3. k个相互独立的hash函数：`h1, h2, ..., hk`

> 可以用$h_{i}(x)=MD5(x+i)$或者$h_{i}(x)=MD5(x|i)$实现k个无关hash函数

需要插入key时，对`key = a`时，经过k个hash函数后得到k个数`h1(a), h2(a), ..., hk(a)`，此时将`bitmap`中对应的位置1。

![往布隆过滤器插入新key](https://raw.githubusercontent.com/oliverdding/imaw.io/main/inserting-key-into-bloom-filter.drawio.svg)

需要查询key时，同样经过k个hash函数得到k个体数，查询`bitmap`中对应位，若全为1，则key**可能**存在；若任意位为0，则**一定不**存在。（这里可以利用短路减少消耗，若出现0则直接返回不存在）

当key越来越多，`bitmap`中为1的位越多，对于某个不存在的key，k个hash函数得到的坐标对应`bitmap`中的值都为1的概率越大，此时就发生false positive。

### 例子

为了直观描述布隆过滤器如何加速查询的，我们用数据库场景来描述。

我们对表中某个字段建立bloom filter，假设有10位的bitmap如下所示：

| 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
| - | - | - | - | - | - | - | - | - | - |
| 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |

我们使用Fnv和Murmur这两个hash函数。

首先插入字符串`hello`，计算`Fnv=6`，`Murmur=0`，于是bitmap如下所示：

| 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
| - | - | - | - | - | - | - | - | - | - |
| 1 | 0 | 0 | 0 | 0 | 0 | 1 | 0 | 0 | 0 |

接着插入字符串`world`，计算`Fnv=7`，`Murmur=1`，于是bitmap如下所示：

| 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
| - | - | - | - | - | - | - | - | - | - |
| 1 | 1 | 0 | 0 | 0 | 0 | 1 | 1 | 0 | 0 |

此时我们查询`charmer`是否存在，计算`Fnv=12`，`Murmur=10`，bitmap中存在0，因此**不存在**。

再查询`hello`是否存在，计算`Fnv=6`，`Murmur=0`，bitmap中都为1，因此**可能存在**，需要访问磁盘，得知真的存在。

再查询`zaslf`是否存在，计算`Fnv=7`，`Murmur=1`，bitmap都为1，因此**可能存在**，需要访问磁盘，得知它不存在。这次磁盘访问就是false positive带来的坏影响，多余的磁盘访问。

![布隆过滤器加速查询](https://raw.githubusercontent.com/oliverdding/imaw.io/main/bloom-filter-speed-up-answers.drawio.svg)

## 数学推导

**k取何值时，false positive发生概率最小，此时概率为？**

由于hash函数“选择”`bitmap`的位置是随机的，因此对于长度为m的`bitmap`而言，插入一个值时某个位被置1、0的概率分别是$P(1) = \frac{1}{m}$、$P(0) = 1 - \frac{1}{m}$

当k个hash函数置位后，对于任意一个位，仍然是0的概率是$$P'(0)=P(0)^{k}=(1 - \frac{1}{m})^{k}$$

对于n个key插入后，对于任意一个位，仍然是0的概率是$$P''(0)=P'(0)^{n}=(1 - \frac{1}{m})^{kn}$$

由于有自然对数e的计算公式$$\lim_{x \to \infty}{(1-\frac{1}{x})^{-x}}=e$$

我们可以**近似**计算$P''(0)$得到$$P''(0)=((1-\frac{1}{m})^{-m})^{-\frac{kn}{m}}\approx{}e^{-\frac{kn}{m}}$$

因此，对于任意一个位，其被置1的概率是$$P''(1)=1-P''(0)\approx{}1-e^{-\frac{kn}{m}}$$

当alse positive发生时，是指它的k个hash函数得到的位置都为1。由于k个hash函数相互独立，我们可以计算出alse positive的概率为$$\varepsilon=(1-(1-\frac{1}{m})^{kn})^{k}\approx{}(1-e^{-\frac{kn}{m}})^{k}$$

由于m、n是用户给定的值，唯一变量就是k，我们需要指定k的值去最小化alse positive概率。

令$p=e^{-\frac{kn}{m}}$以及自然对数公式，我们有$$\varepsilon=(1-p)^{k}=e^{kln(1-p)}$$

此时变更为最小的$g=kln(1-p)$

无中生有下得到$$g=kln{}(1-p)=-\frac{m}{n}ln(e^{-\frac{kn}{m}})ln(1-p)=-\frac{m}{n}ln(p)ln(1-p)$$

根据对称性原理得知当$p=\frac{1}{2}$时g取得最小，此时有$$k=\frac{m}{n}ln(2)$$

插入回$$\varepsilon=(1-p)^{k}$$得到alse positive的最小值为$$\varepsilon_{min}=(\frac{1}{2})^{k}\approx{}(0.6185)^{\frac{m}{n}}$$

## 应用

那么如何在项目中使用bloom filter呢？

### 已知$n$和$\varepsilon$

$n$由系统输入决定，$\varepsilon$则是开发者平衡内存使用和性能的选择，因此可以通过传入这两个变量自动计算剩下的值。

1. 根据公式$m=\lceil{}-\frac{nln(\varepsilon)}{(ln(2))^2}\rceil$计算m的值
2. 根据公式$k=\lceil{}-log_2(\varepsilon)\rceil$计算k的值

## 缺陷

基础bloom filter有着自己的局限性，因此出现了许多变种，它们为了特殊场景作出不同的定制，增加一定开销换取更多功能：

* 无法删除元素：计数布隆过滤器
* 无法扩展：[可扩展布隆过滤器](/posts/bloom-filter-2/)

## 参考

- [1]: Space/Time Trade-offs in Hash Coding with Allowable Errors
- [2]: Summary Cache: A Scalable Wide-Area Web Cache Sharing Protocol 
- [3]: Network Applications of Bloom Filters: A Survey
- [4]: Bloom Filters by Example. https://llimllib.github.io/bloomfilter-tutorial/
