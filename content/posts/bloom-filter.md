+++
title="布隆过滤器"
description="用于判断某个元素是否在集合中的一个简单、高效、内存占用低的数据结构"
date=2022-02-09T17:13:10+08:00

[taxonomies]
categories = ["DataStructures"]
tags = ["big data", "data structure"]

[extra]
toc = true
comments = true
+++

布隆过滤器（Bloom Filter）是一种类似于哈希表的数据结构，但相比于后者空间利用率更高。

使用布隆过滤器查找key时，返回值可能情况有两种：

1. key**不**存在
2. key**可能**存在

## 原理

布隆过滤器由以下三部分构成：

1. n个key
2. m位的位图：`bitmap`
3. k个无关的hash函数：`h1, h2, ..., hk`

> 可以用$h_{i}(x)=MD5(x+i)$或者$h_{i}(x)=MD5(x|i)$实现k个无关hash函数

需要插入key时，对`key = a`时，经过k个hash函数后得到`h1(a), h2(a), ..., hk(a)`

此时将`bitmap`中对应的位置1。

![布隆过滤器插入新key示意图](/images/布隆过滤器插入新key.drawio.svg)

当key越来越多，`bitmap`中为1的位越多，对于某个不存在的key，k个hash函数得到的坐标都为1的可能越大，此时就发生false positive。

那么k取何值时，false positive发生概率最小，此时概率为？

## 数学证明

对`bitmap`随机置位后，对于任意一个位，其被置0、1的概率分别是$P(0) = 1 - \frac{1}{m}$、$P(1) = \frac{1}{m}$

对于k个hash函数的置位后，对于任意一个位，仍然是0的概率是$$P'(0)=P(0)^{k}=(1 - \frac{1}{m})^{k}$$

对于n个key插入后，对于任意一个位，仍然是0的概率是$$P''(0)=P'(0)^{n}=(1 - \frac{1}{m})^{kn}$$

由于有自然对数e的计算公式$$\lim_{n \to \infty}{(1-\frac{1}{x})^{-x}}=e$$

我们可以近似计算$P''(0)$得到$$P''(0)=(1-\frac{1}{m})^{kn}\approx{}e^{-\frac{kn}{m}}$$

因此，对于任意一个位，其被置1的概率是$$P''(1)=1-P''(0)\approx{}1-e^{-\frac{kn}{m}}$$

当alse positive发生时，是指它的k个hash函数得到的坐标都为1。由于k个hash函数相互独立，我们可以计算出alse positive的概率为$$f=(1-(1-\frac{1}{m})^{kn})^{k}\approx{}(1-e^{-\frac{kn}{m}})^{k}$$

由于m、n是用户给定的值，唯一变量就是k，我们需要指定k的值去最小化alse positive概率。

另$p=e^{-\frac{kn}{m}}$，我们有$$f=(1-p)^{k}=e^{kln(1-p)}$$

此时变更为最小的$g=kln(1-p)$

无中生有下得到$$g=kln{}(1-p)=-\frac{m}{n}ln(e^{-\frac{kn}{m}})ln(1-p)=-\frac{m}{n}ln(p)ln(1-p)$$

得知当$p=\frac{1}{2}$时g取得最小，此时有$$k=ln(2\frac{m}{n})$$

插入回$$f=(1-p)^{k}$$得到alse positive的最小值为$$f_{min}=(\frac{1}{2})^{k}\approx{}(0.6185)^{\frac{m}{n}}$$
