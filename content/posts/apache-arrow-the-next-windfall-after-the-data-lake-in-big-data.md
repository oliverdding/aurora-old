+++
title = "Apache Arrow: 大数据在数据湖后的下一个风向标"
description = "Apache Arrow的产生、现状和发展"
date = 2021-11-18T10:59:50+08:00


[taxonomies]
categories = ["DataStructures"]
tags = ["big data", "apache", "data structure"]

[extra]
toc = true
comments = true
+++

## 介绍

根据[官方文档](https://arrow.apache.org/overview/)介绍，Arrow是

> A language-independent columnar memory format for flat and hierarchical data, organized for efficient analytic operations on modern hardware like CPUs and GPUs. The Arrow memory format also supports zero-copy reads for lightning-fast data access without serialization overhead.

一句话概括，Arrow用于系统间高效交互数据的组件。

### Arrow的核心能力

Arrow本身不是一个存储、执行引擎，它只是一个交互数据的基础库。比如可以用于以下组件

* SQL执行引擎 (e.g., Drill and Impala)
* 数据分析系统 (e.g., Pandas and Spark)
* 流和队列系统 (e.g., Kafka and Storm)
* 存储系统 (e.g., Parquet, Kudu, Cassandra and HBase)

## 背景

> 每个事物的产生发展都有其历史原因，如果抛开目的去“学习”，犹如竹篮子打水-一场空
>
> <p align="right">- 我说的 ;)</p>

让我们回到2008年，故事从那开始...

### 起因

Wes McKinney在2008年开启了Pandas项目，这个python中分析、操作数据的瑞士军刀。紧接着在2014年，Wes加入Cloudera公司，并着手研究如何让python可以“插入”所有的大数据组件和数据库，但是每个系统都有自己操作数据的方式，于是：

> "Oh my gosh, I'm going to have to write a dozen different data converters to marshal data, convert data between Pandas each of these data processing systems from Spark, to Impala, to different file formats and HDFS, so it was basically this overwhelming problem."
>
> <p align="right">- Wes McKinney</p>

除此之外，在大数据科学领域，dataframe的概念随处可见，每个框架都将datafrme作为高层定义，代表一个表、一系列API...但是其底层的实现天壤悬隔，Wes完全无法复用代码。

**无法共享数据**、**无法共享代码**这两个大难题暂时困住了Wes。

### 发展

Wes开始设计一种table middleware，作为不同组件交换数据的中间层，一种表接口的标准(standardized table interface)。

接着来到2015年，Wes团队遇到了Jacques和Apache Drill社区的小伙伴们，两伙人不谋而合，开始了合作。

由于业界没有统一规范的定义，他们合作的首个项目就是设计出了一个内存表视图的标准，并在不同语言都给出实现以证明可以在不同语言中共享数据，也就是说，你可以高效地将数据从Java到C++，或者Python。

自此，arrow项目创立。

在项目早期，最重要的是设计出一套与语言无关的内存表结构，并一定要方便分析处理。除此之外，还需要将各种格式、类型的数据转换、转出为这个标准格式的库。最后，还需要一个计算处理的库，以便于直接基于arrow进行快速数据分析处理。

> "An important thing to remember about the project is that it's front-end agnostic. So it's not a new data frame library, like Pandas. It's not a new database."
>
> <p align="right">- Wes McKinney</p>

此外，Wes在和Apache Impala团队合作的时候，发现Impala的代码中有大量和pandas做相似事情的片段，比如CSV序列化、反序列化的，I/O子系统，自己的查询引擎，甚至自己的前端。在有了这样一个语言无关的内存数据格式，他们开始思考如何避免重复代码。

## 实现

故事讲完了，现在让我们一起来探索下arrow的设计。

面对不同语言、不同大数据组件之间的差异，首先我们肯定需要一个中间的表示来避免我们的后端直面差异，也就是前文提到的语言无关的内存表视图，这里就有一个必须挖掘的点，为了批量数据分析，我们应当选择**列式存储**。

![列存表查询](https://i.loli.net/2021/11/13/BNGPznF9xUy4qMi.png)

使用列存的方式不仅减少了扫描内存的page数，还可以利用现在计算机SIMD(Single Instruction, Multiple Data)指令进行加速。

---

扩展阅读 - [Daniel Abadi的实验](http://dbmsmusings.blogspot.com/2017/10/apache-arrow-vs-parquet-and-orc-do-we.html)

Daniel在亚马逊的EC2 t2.medium机器上创建了一个有60,000,000行数据的内存表。表由6个int32列组成，整个表大概由1.5GB。他创建了行表和列表两个实例，并对两种表进行简单地filter某个值。

在未开CPU优化的情况下，得到结果：

![无SIMD](https://i.loli.net/2021/11/13/wjkzO3cfR1BAdiE.png)

行表和列表查询耗时相差无几。对于行表，每行都需要扫描，即使只使用到第一列；对于列表则只需要扫描第一列，按理说列表应该是行表的6倍快，但是在这个实验中由于CPU是瓶颈，而不是内存发往CPU的数据。

但是开启SIMD后，结果如下：

![开SIMD](https://i.loli.net/2021/11/13/ysv8F1SCMaQHEOw.png)

SIMD可以同时比较多个数值（这里是4个数，差不多3倍快），减少打乱流水线的情况

---

现在我们可以继续考虑如何设计语言无关的内存表结构了

![直接IPC](https://i.loli.net/2021/11/13/Ca6DOkIohglrXx4.png)

Arrow需要作为通用的传输结构

![通过arrow交互](https://i.loli.net/2021/11/13/uJ5fAIYCXZK9eND.png)

可是代码共享该如何实现呢？Arrow不应该是json、protobuf之流，后者适用于磁盘层面的数据存储交互。Arrow应当作为各个语言、组件中的一种数据格式库，应该是运行时的数据存储交互！直接可以操作数据，存取、计算：

![数据操作](https://i.loli.net/2021/11/13/pGxX76nHLvmNPYy.png)

## Arrow列格式

> :construction: 本节内容翻译整理自apache/arrow代码仓库中[Arrow Columnar Format规范](https://github.com/apache/arrow/blob/master/docs/source/format/Columnar.rst)。

Arrow列格式包含三部分：与语言无关的内存数据结构规范、元数据序列化以及一个用于序列化和通用数据传输的协议。

该列格式支持：

* 顺序访问的数据
* O(1)的随机读写
* 支持SIMD，向量化操作友好
* 可重新定位而无“pointer swizzling”问题，允许在共享内存中zero-copy

---

扩展阅读 - [pointer swizzling](https://en.wikipedia.org/wiki/Pointer_swizzling)

简单来说，内存中指针所指向的地址在写入磁盘（序列化）和从磁盘载入指针数据（反序列化）时，需要通过某种方式（swizzling和unswizzling）来使得指针存储的地址信息有效。

扩展阅读 - [零拷贝](https://en.wikipedia.org/wiki/Zero-copy)

zero-copy（零拷贝）不是指真的没有拷贝了，而是说减少了不必要的数据拷贝与上下文切换（系统调用）。比如正常情况下用户态进程希望从磁盘中读取数据并写入socket，此时需要数据流经过磁盘->系统态内存->用户态内存->系统态内存->socket，发生了两次系统调用(磁盘的read()和写入socket的write())。使用系统提供的零拷贝函数(比如sendfile())则可以缩减为磁盘->系统态内存->socket。

---

在Arrow中，最基本的结构是array(或者叫vector，是由一列相同类型的值组成，长度必须已知，且有上限；换个常见的叫法是field，字段)，每个array都有如下几个部分组成：

* 逻辑上的数据类型（记录array类型）
* 一列缓冲区（存放具体数字、null）
* 一个长度为64位带符号的整数（记录array长度，也可以是32位）
* 另一个长度为64位的带符号的整数（记录null值的数量）
* （可选）字典（用于字典编码的array）

Arrow还支持嵌套array类型，其实就是一列array组成，它们叫做子array(child arrays)。

### 物理内存布局

每一个逻辑类型都有一个定义明确的物理布局，Arrow定义了如下物理布局：

* Primitive(fixed-size)：用于存放具有相同长度的数值
* Variable-size Binary：用于存放长度可变的数值。支持32位和64位的长度编码
* Fixed-size List：嵌套类型，但是每个子array长度必须相同
* Variable-size List：嵌套类型，每个子array长度可以不一致。支持32位和64位的长度编码
* Struct：嵌套类型，由一组长度相同的命名子字段组成，但子字段的类型可以不一致。
* Spare和Dense Union：嵌套类型，但是只有一组array，每个数值的类型是子类型集合之一
* Null：存放一组null值，逻辑类型只能是null

#### 布局例子

本小节以Fixed-size Primitive Layout为例子讲述Arrow最基础的内存布局。

如前文所述，Primitive类型的数值槽长度相同，只能存放固定长度的数值，可以是字节或者比特。

放到具体内存布局上，本类型包含一个连续的内存缓冲区，总大小则是槽宽\*长度（对于比特的槽宽，则需要四舍五入到字节）。

给出文档中一个Int32 Array的例子：

```
[1, null, 2, 4, 8]
```

会这样表示：

```
* Length: 5, Null count: 1
* Validity bitmap buffer:

  |Byte 0 (validity bitmap) | Bytes 1-63            |
  |-------------------------|-----------------------|
  | 00011101                | 0 (padding)           |

* Value Buffer:

  |Bytes 0-3   | Bytes 4-7   | Bytes 8-11  | Bytes 12-15 | Bytes 16-19 | Bytes 20-63 |
  |------------|-------------|-------------|-------------|-------------|-------------|
  | 1          | unspecified | 2           | 4           | 8           | unspecified |
```

其中有效性位图是用于记录每个值槽是否为空的。具体看[规范](https://arrow.apache.org/docs/format/Columnar.html#validity-bitmaps)。

剩下的布局都在Primitive布局上变化而来，具体看规范。

#### 布局使用的缓冲区

Arrow的几种物理布局用到的缓冲区如下表所示：

| **Layout Type**    | **Buffer 0** | **Buffer 1**   | **Buffer 2** |
|:------------------:|:------------:|:--------------:|:------------:|
| Primitive          | validity     | data           |
| Variable Binary    | validity     | offsets        | data         |
| List               | validity     | offsets        |
| Fixed-size List    | validity     |                |
| Struct             | validity     |                |
| Sparse Union       | type ids     |                |
| Dense Union        | type ids     | offsets        |
| Null               |              |                |
| Dictionary-encoded | validity     | data (indices) |

Arrow如何实现O(1)读写的呢？

所有的物理布局底层都是用数组存储数据，并且会根据层级嵌套建立offsets bitmap，当然就实现了O(1)的读写速度了。

### 逻辑类型

[Schema.fbs](https://github.com/apache/arrow/blob/master/format/Schema.fbs)定义了Arrow支持的逻辑类型，每种逻辑类型都会对应到一种物理布局。

### 序列化与IPC

列式格式序列化时最原始的单位是"record batch"(也就是一个表，table啦)。一个record batch是一组有序的array的集合，被称为record batch的字段(fields)。每个字段(field)有相同的长度，但是字段的数据类型可以不一样。record batch的字段名、类型构成了它的schema。

本节描述一个协议，用于将record batch序列化为二进制流，并可以无需内存拷贝重构record batch。

序列化时会分为这三部分：

- Schema
- RecordBatch
- DictionaryBatch

这里我们只提及前两个。

![布局](https://i.loli.net/2021/11/16/ptCPLjx7bcWOXmi.png)

一个schema message和多个record batch message就能完整表示一个record batch。其中schema message存储表结构，record batch message存储字段metadata和字段值。

值得注意的是，record batch message包含实际的数据缓冲区、对应的物理内存布局。

然后问题又来了，Arrow为何无需pointer-swizzling即可实现流与数据转换的呢？答案就是message的metadata中存储了每个缓冲区的位置和大小，因此可以字节通过指针计算来重建Array数据结构，同时还避免了内存拷贝。

于是定义IPC流格式：

```
<SCHEMA>
<DICTIONARY 0>
...
<DICTIONARY k - 1>
<RECORD BATCH 0>
...
<DICTIONARY x DELTA>
...
<DICTIONARY y DELTA>
...
<RECORD BATCH n - 1>
<EOS [optional]: 0xFFFFFFFF 0x00000000>
```

由于这部分比较“定义”，本文不展开讲，更详细请看[规范](https://arrow.apache.org/docs/format/Columnar.html#ipc-streaming-format)。

## Arrow Flight

近段时间Arrow最大的变化就是添加了Flight，一个通用C/S架构的高性能数据传输框架。Flight基于gRPC开发，从最开始重点就是优化Arrow格式数据。

Flight的具体细节请看[官方文档](https://arrow.apache.org/blog/2019/10/13/introducing-arrow-flight/)。这里只介绍它的优势：

- 无序列化/反序列化：Flight会直接将内存中的Arrow发送，不进行任何序列化/反序列化操作
- 批处理：Flight对record batch的操作无需访问具体的列、记录或者元素
- 高并发：Flight的吞吐量只收到客户端和服务端的吞吐量以及网络的限制
- 网络利用率高：Flight使用基于HTTP/2的gRPC，不仅是快

[官方给出的数据](https://www.dremio.com/is-time-to-replace-odbc-jdbc/)是Flight的传输大约是标准ODBC的20-50倍。

对每个batch record平均行数256K时，在单节点传输时的性能对比（因为flight多节点时可以平行传输数据流）：

![性能对比](https://i.loli.net/2021/11/16/EOrNiXqY9Le2jTy.png)

### 使用场景

最过经典的非[PySpark](https://arrow.apache.org/blog/2017/07/26/spark-arrow/)莫属，此外还有[sparklyr](https://arrow.apache.org/blog/2019/01/25/r-spark-improvements/)。

另外，ClickHouse也有计划实现Arrow Flight的server端，一旦落地可用，spark与clickhouse交互就可以抛弃3G网般的JDBC了~

## 总结

本文从Arrow立项的背景入手，再到Arrow实现所需的设计，最后到Arrow具体columnar格式定义，介绍了Arrow的各种相关概念。最后补上一张图作为Arrow的优点、限制的总结：

![总结](https://i.loli.net/2021/11/14/i9hQ5wd4sj6y8Tf.png)

## 参考

1. Wes和Jacques的视频访谈: [Starting Apache Arrow](https://www.dremio.com/starting-apache-arrow/)
2. Arrow起名投票: [Vector Naming Discussion](https://docs.google.com/spreadsheets/d/1q6UqluW6SLuMKRwW2TBGBzHfYLlXYm37eKJlIxWQGQM/edit#gid=0)
3. 思路来源: [伴鱼技术团队](https://tech.ipalfish.com/blog/2020/12/08/apache_arrow_summary/)
4. [Arrow Columnar Format](https://github.com/apache/arrow/blob/master/docs/source/format/Columnar.rst)
5. [Arrow FAQ](https://arrow.apache.org/faq/)
