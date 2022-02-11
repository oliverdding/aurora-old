+++
title = "论文阅读笔记：RDD"
description = "Matei Zaharia, Mosharaf Chowdhury, Tathagata Das, Ankur Dave, Justin Ma, Murphy McCauley, Michael J. Franklin, Scott Shenker, Ion Stoica. Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing"
date = 2021-10-12T17:31:07+08:00

[taxonomies]
categories = ["Notes"]
tags = ["paper reading", "note", "spark", "apache", "big data"]

[extra]
toc = true
comments = true
+++

## Abstract

RDD起源于现有的大数据框架对

1. 迭代算法: 数据复用
2. 交互式数据
3. 挖掘: 数据共享

支持过差

## Introduction

新提出的Pregel、HaLoop虽然依次针对性提出解决方案，但是方案缺乏统一的抽象接口，无法推广。

于是，研究人员提出RDD，支持大量场景下内存复用。

> Resilient Distributed Datasets(RDDs): RDDS are fault-tolerant, parallel data structures that enable users explicitly persist intermediate results in memory, control their partitioning to optimize data placement, and manaipulate them using a rich set of operations.

最大难题是设计接口，如何设计错误容忍度高的接口呢？参考现有设计，DSM、KV-stores、databases和Piccolo，他们提供细粒度内存接口，导致想要支持错误恢复，必须：

1. 跨设备备份数据
2. 跨设备日志更新

这对于大数据应用而言太昂贵了，于是，RDD被设计为粗粒度内存模型接口，提供transformations接口对不同数据执行某些操作操作。此时，备份变得简单，只要记录“血统”(每个RDD的transformations流程)。如果某个RDD的分区丢失了，完全可以只恢复这个分区的RDD数据。

## RDD

### RDD概述

RDD是只读的数据分区集合。每个RDD只有三个来源：

1. 从内存对象集合生成
2. 从外存数据生成
3. 从现有RDD转换

用户拿到RDD不一定真实存在，可能只是这个RDD的“血统”。

> A program cannot reference an RDD that it cannot reconstruct after a failure.

### Spark编程模型

spark程序往往起始于通过transformations创建RDD，终止于通过actions处理RDD返回数值结果或者写入存储。

RDD不一定是实体的，在action被运行前才被计算，因此可以排排站，吃果果。

### RDD对比DSM的优点

Distributed Shared Memory(DSM): 另一种分布式框架，细粒度内存共享。

RDD的优点：

1. 更高效的错误容忍度
2. 得益于RDD不可变，可以运行时同时备份数据
3. 任务可以被调度到离数据物理更近的节点上
4. 当内存不够时RDD可以优雅缩减RDD数量规模

### 不适合于RDD的场景

RDD只适合于**批处理**应用。

RDD不适合于异步细粒度更新共享状态的应用。

## Spark编程接口

开发人员主要写`driver program`，用于连接集群中的workers。

### Spark中RDD的操作

* `transformations`: 定义RDD的懒操作
* `actions`: 执行计算，返回结果或者写入存储

## RDD实现

提出五个需要实现的接口。

* partitions()
* preferredLocations(p)
* dependencies()
* iterator(p,parentIters)
* partitioner()

依赖分位窄依赖和宽依赖。

为什么要分窄依赖？

1. 可以管道执行
2. 恢复更简单
