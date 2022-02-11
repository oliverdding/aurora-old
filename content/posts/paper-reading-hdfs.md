+++
title = "论文阅读笔记：HDFS"
description = "Konstantin Shvachko, Hairong Kuang, Sanjay Radia, Robert Chansler. The Hadoop Distributed File System"
date = 2021-09-17T13:17:46+08:00

[taxonomies]
categories = ["Notes"]
tags = ["paper reading", "note", "spark", "apache", "big data"]

[extra]
toc = true
comments = true
+++

<mark>注意: 本文涉及到的专有名词将直接使用英文, 便于理解记忆</mark>

<mark>注意: 本文面向hadoop 2版本, 其它版本差异并不涉及</mark>

Hadoop是什么? SAS给出的描述[^6]是:

> Hadoop is an open-source software framework for storing data and running applications on clusters of commodity hardware. It provides massive storage for any kind of data, enormous processing power and the ability to handle virtually limitless concurrent tasks or jobs.

不难看出, 其中commodity hardware, massive storage和enormous processing power就是Hadoop的重要特点. 而The Hadoop Distributed File System(HDFS)作为Hadoop的核心子项目之一, 是Google File System(GFS)的实现, 为分布式计算提供数据存储和管理的功能.

## 背景

讲到HDFS就不得不提及GFS. 随着业务发展, Google公司对数据存储处理的需要日益增长, 便设计出GFS, 实现了在廉价的商品机型上高容错地存储数据, 并提供了高计算性能, 满足了Google公司的需要[^5]. HDFS正是基于Google公司在2000年左右完成的GFS而改进实现的[^1], 主要是改为通用文件系统(可以支持更多的存储系统而不只是Google的), 并为了内存一致性和性能, 修改了写入模型和写入流程, 其他几乎和GFS一致.

HDFS将metadata和data分开存放, 像其它的同类文件系统比如PVFS[^4][^7], Lustre[^3]和GFS, HDFS将metadata存储于一个叫Name Node的专用服务器上, 将data存储于叫Data Node的服务器上. 所有的服务器都通过一个基于TCP的专用协议进行互联通信.

不同于Lustre和PVFS, HDFS的Data Node并不使用诸如RAID的数据保护策略, 而是仿照GFS, 将文件冗余存放于不同的Data Node. 这种方式除了提高数据的安全性, 还提高了系统的可用带宽, 因为随着文件冗余存储于不同的节点, 对于需要数据计算的软件而言, 数据正好在本地Data Node的概率更高.

## 目的

1. 硬件错误时的高容错性

   对于面向成千上万的廉价机型的集群而言, 机器的硬件错误是非常常见的. HDFS集群中每个节点都存储着文件系统的部分数据, 因此, 为了数据的安全性, HDFS需要能检测硬件错误, 并快速地, 自动地从错误中恢复.

2. 流式数据读写

   许多应用需要流式数据访问, 但它们并不是运行在通用文件系统上的通用应用程序. 因此, HDFS面向批处理设计而不是普通用户交互访问, 也就是说, 重点是数据访问的高吞吐量而不是数据访问低延迟. 为了实现这一点, HDFS放宽了部分POSIX的要求.

3. 存储大数据集

   存储在HDFS的文件可能是以GB, TB为单位的文件, HDFS provides high aggregate data bandwidth and can scale to hundreds of nodes in a single cluster.[^2]

4. 保证内存一致性

   为了保证内存一致性, HDFS对读写访问模型做了简化, 区别于GFS, HDFS仅仅支持一个写入者和多个读入者. 并且一个文件一旦创建, 写入并关闭, 除了append和truncate外不能执行其他操作. 通过这些约束简化了内存一致性模型, 并支持高吞吐量数据访问.

5. 更强的可移植性

   基于Java的HDFS, 不必多说.

## 架构

这里放上一个视频, 比较生动形象, 先看一篇有个大概的印象再阅读文章.

[![Understanding HDFS using Legos](https://res.cloudinary.com/marcomontalbano/image/upload/v1613722386/video_to_markdown/images/youtube--4Gfl0WuONMY-c05b58ac6eb4c4700831b2b3070cd403.jpg)](https://youtu.be/4Gfl0WuONMY "Understanding HDFS using Legos")

下图是官网的HDFS架构图解.

![HDFS Architecture](https://hadoop.apache.org/docs/r2.10.1/hadoop-project-dist/hadoop-hdfs/images/hdfsarchitecture.png)

### Name Node

HDFS的namespace是文件和目录的层次结构. 文件和目录在Name Node中以inode存储, 记录着像是权限, 修改访问时间, namespace和大小等等属性. 文件被切分为大的块(默认是128MB, 可以每个文件单独指定), 并且文件的每个块被独立冗余备份在不同的Data Node上(一般是3份, 同样也可以每个文件单独指定). Name Node维护namespace的树状结构, 并保存文件块到Data Node的映射. 

当读取文件时, HDFS client先从Name Node获取存放文件数据块的Data Node, 然后直接从**最近**的Data Node获取数据. 同样的, 当写入数据时, 客户端会要求Name Node指定一组Data Node存储文件块和文件块的副本, 之后以管道的方式向Data Nodes写入数据.

在当前的设计中, 每个集群只有一个Name Node, 但有任意的Data Node和HDFS Client. HDFS将整个namespace存储于RAM中. inode数据和文件块的列表组成了系统的metadata, 被称为image. image被永久存储在本地文件系统, 被称为checkpoint. Name Node同样会存储image的修改日志, 被称为journal. 为了提高耐久性, checkpoint和journal可以冗余备份在不同的服务器. 当重启时, Name Node通过读入checkpoint并执行journal恢复原先的namespace. 块副本的位置可能会改变, 因此checkpoint不包括这部分.

### Data Node

每个块存储于Data Node时, 都由两个文件组成. 第一个包含了实际数据本身, 第二个是块的metadata, 包括校验码(checksum)和版本戳(generation stamp). 包含数据的文件的大小等同于块的真实大小, 不需要像传统文件系统那样需要额外空间以填充到标称块大小. 因此一个文件块若是只有一半的大小, 它就仅仅需要本地文件系统半块的的空间.

在startup阶段, 每个Data Node都会连接到Name Node并执行握手. 握手的目的之一是验证namespace ID和软件版本. 若任意一个没有匹配上, Data Node将会自动关机. 当HDFS文件系统格式化时会生成对应的namespace ID, 之后被存储于集群中所有节点上, 拥有不同的namespace ID的节点不可能在同一集群中, 保证了文件系统的完整性. 软件版本的一致性也是必要的, 因为不同版本的软件会导致数据损坏或丢失, 当更新时, 大集群中总会有节点未能正确关机, 更新, 导致软件版本不对, 这时候就需要关闭这些节点, 等后续手动操作.

刚刚初始化的Data Node, 可以加入任一集群并接受集群的namespace ID. 在握手后, Data Node注册到Name Node上. Data Node也存储着他自己独一无二的storage ID, 作为Data Node的内部标识, 保证了即使重启后, 被分配了不同的IP地址和端口, 仍能被正确识别. 而storage ID时Data Node注册到Name Node时被分配的, 以后**不会再改变**.

Data Node定期发送block report给Name Node, 包含block id, 版本戳和全部块的长度信息. 一经注册就会发送第一个block report. 之后每小时都会发送一次.

在运转过程中, Data Node会发送heartbeat给Name Node以告知操作正在进行并且数据正常. 默认的时间间隔是3秒. 若是10分钟内, Name Node未能收到Data Node的heartbeat信号, 将会认为Data Node离线并且上面的数据都丢失, 便会重新规划块备份.

事实上, heartbeat信号还包含了一些额外的信息, 包括中存储容量, 已使用存储的百分比, 正在处理的数据数量. 这些统计信息被Name Node用来平衡负载和安排空间.

Name Node并不会直接调度Data Node, 而是通过回复heartbeat的方式发送命令. 命令包括:

1. 往其他Data Node备份块
2. 删除本地块备份
3. 重新注册或关闭node
4. 立刻发送block report

### HDFS Client

用户应用程序想要访问HDFS, 必须使用HDFS Client. 和传统文件系统类似, HDFS支持读, 写和删除文件, 以及创建和删除目录. 用户通过路径指定namespace内的文件和目录. 用户完全不用知道细节.

当应用读取文件时, HDFS Client会向Name Node请求存储着文件块的Data Node列表. 然后直接访问Data Node并获取数据. 当HDFS Client写入数据时, 首先请求Name Node选择第一个块的备份Data Node列表, Client会组织一个node-to-node的管道并写入数据. 当第一个块完成后, Client会再次请求Data Node列表, 再以同样的方式写后面的块.

不同于传统文件系统, HDFS提供获取文件块位置的API, 允许应用程序直接将任务运行于目标Data Node上, 减少了不必要的流量消耗. 同时, 允许应用程序设置文件的冗余因子, 默认情况下是3. 对于重要的文件和经常访问的文件, 可以设置为更高的值, 以保证数据安全和提高读带宽.

### Image和Journal

正如前文所说, image就是当前metadata的版本, 而checkpoint是写入到外存的image, 永远落后于当前RAM中的image, 而journal则记录着checkpoint到当前image之间的操作. checkpoint不会被修改, 只会在重启时被完整替换. 在startup阶段, Name Node从checkpoint中读取image, 并对其执行journal的操作, 以恢复上次的image. 当完成后且开始运作之前, 新的checkpoint和空白的journal文件会被回写.

若是checkpoint或journal文件丢失或者损坏, namespace的信息就丢失了. 为了保护这重要的信息, HDFS可以存储checkpoint和journal文件在不同的存储目录.

> 最佳实践: 将这两个文件存储在不同的卷, 甚至远程NFS服务器.

Name Node是多线程服务器, 可以同时处理多个Client的请求. 于是, 将事务写入外存成为了整个系统的瓶颈, 因为一个线程在同步地写入时, 其他线程只能等待. 为了进一步优化, Name Node通过批处理的方式写入事务. 当一个线程请求flush-and-sync操作时, 所有在等待处理的提交会被同时写入. 剩下的线程只需要去检查自己的事务是否被保存, 而不是都去flush-and-sync.

### Checkpoint Node

Name Node在HDFS中, 除了为Client提供服务外, 还可以作为Checkpoint Node和Backup Node.

Checkpoint Node定时的合并现有的checkpoint和journal, 创建新的checkpoint和空的journal. Checkpoint Node通常是由运行在其他主机, 因为它必须要有和Name Node相同的内存需求. 它会从Name Node下载最新的checkpoint和journal, 合并, 然后回写.

这种方式减少了startup阶段的耗时, 因为减少了journal文件的大小.

> 最佳实践: 每日更新一个checkpoint.

### Backup Node

这是HDFS的新特性. 这部分不详细说, 类似双机热备份一样, 只是只读的备份.

### 更新与文件系统快照

在软件更新过程中, 最容易出现数据损坏. 为了降低这种风险, HDFS拥有快照功能. 快照允许管理员永久地保存当前文件系统的状态, 因此当更新异常导致数据丢失损坏时可以恢复到快照的状态.

在系统开启时, 可以选择开启快照功能(只能同时保存一个快照). 当要求快照时, Name Node首先读取checkpoint和journal文件, 并在内存中合并, 然后将新checkpoint和空的journal写入新的路径, 旧的checkpoint和journal仍然未改变.

## 数据操作

数据操作主要是文件的读取写入以及副本的放置策略, 算是HDFS的核心部分.

### 文件写入

前文已经提到, 当一个应用创建文件写入数据并关闭后, 内容不再可以改变, 除了append和truncate.

HDFS实现single-writer-multiple-reader模式. 也就是说, 当一个HDFS Client打开一个文件并写入数据时, 会暂时独占这个文件的, 我们称之为lease, 其它的client都不再可以写入数据. 写入数据的client通过heartbeat信号更新自己对文件的lease. 当文件关闭, lease被回收. lease的期限受到软限制(soft limit)和硬限制(hard limit). 在软限制过期之前, 写入数据的client独占这个文件. 当软限制过期, 而client未能关闭文件或者更新自己的lease, 另一个client就可以抢占lease. 当硬限制过期(一小时), 而client未能更新lease, HDFS会当作client已经退出, 并自动关闭文件, 回收lease. 注意, lease是写入独占, 其它client仍然可以读取文件.

HDFS的文件是由block组成的. 当需要新的block时, Name Node将分配一个独一无二的block ID并决定一组Data Node去存放block以及它的副本. Data Node之间会组织管道, 它们的顺序会自动最优化, 降低带宽损耗. 数据被以数据包的方式序列化传输. 当第一个数据包被填满(一般是64KB), 数据包会被推送到管道. 下一个数据包可以在收到ACK(由管道最后一个Data Node回复)之前推出(就是TCP). 已发送未收到ACK的包的数量被client的滑动窗口限制(这里是TCP的滑动窗口).

数据写入HDFS文件后, 在文件被关闭前, HDFS并不保证文件立马可见. 若是应用程序需要边写边可读, 可以通过`hflush`操作: 当前数据包被立马发送到管道, 并同步等待管道中的ACK回复再继续写入.

![HDFS example](https://i.loli.net/2021/09/17/gHwyJF6TKZ8ReNs.png)

上图是HDFS写入的例子.

### 文件读取

在一个由成千上万node组成的集群中, 错误(通常是存储错误)经常发生. 存储于Data Node上的块可能由于内存错误, 硬盘错误或者网络错误导致损坏. 为了避免这些, HDFS生成并存储每个数据block的checksum. checksum被client在读取数据时校验. 当client创建一个HDFS文件时, 同时会计算每个块的checksum并和数据一同发送到管道. 当HDFS读取文件时, 数据block文件和checksum会被同时发送给client, client将会计算校验. 若出错, client会通知Name Node数据算坏, 然后从另一个Data Node获取该block.

当client打开读取一个文件, client将获取块的存储列表, 并选择最近的Data Node读取数据. 当读取失败, 就会尝试下一个. 可能会有疑问, 什么时候读取会失败呢? 有三个原因:

1. 目标Data Node挂了.
2. 目标Data Node并没有目标block了
3. 目标block校验不通过

### 副本管理

Name Node还需要保证每一时刻每个块都有足够数量的副本. 每次收到block report, Name Node都会检测是否有块的副本数量大于少于预设值. 当副本数量大于预设值, Name Node会选择删除副本, 依据尽量不减少拥有副本的rack(这里的rack, 我理解为物理上放置在同一地点, 网络上属于同一网络的群组)数量. 当副本数了小于预设值, Name Node会将它放到复制优先队列(replication priority queue). 一个后台进程会不断扫描该优先队列的队头并决定哪个Data Node放置副本.

这部分细节非常多, 写下会导致篇幅过大, 请读者自行深入.

可以参看本文的基础论文: Konstantin Shvachko, Hairong Kuang, Sanjay Radia, Robert Chansler. "The Hadoop Distributed File System"

## 总结

综上, HDFS是基于GFS的开源分布式文件系统, 具有高容错性, 可以部署在廉价不可靠的机器上. 适合批处理, 大量数据集存储处理与流式文件访问.

当然HDFS也有着自己的局限, 世界上没有十全十美的事情, 就好像为了简化内存一致性模型HDFS不得不使用one-writer的模式, 又好像为了提高吞吐量不得不放松POSIX标准.

1. 不支持低延迟访问
2. 不适合小文件存储
3. 不支持并发写入
4. 不支持修改

## 参考

[^1]:Google File System (GFS) and Hadoop Distributed File System (HDFS). http://stg-tud.github.io/ctbd/2017/CTBD_06_gfs-hdfs.pdf . p5
[^2]:HDFS Architecture. https://hadoop.apache.org/docs/r2.10.1/hadoop-project-dist/hadoop-hdfs/HdfsDesign.html
[^3]:Lustre File System. http://www.lustre.org
[^4]:P. H. Carns, W. B. Ligon III, R. B. Ross, and R. Thakur. “PVFS: A parallel file system for Linux clusters,” in Proc. of 4th Annual Linux Showcase and Conference, 2000, pp. 317–327.
[^5]:S. Ghemawat, H. Gobioff, S. Leung. “The Google file system,” In Proc. of ACM Symposium on Operating Systems Principles, Lake George, NY, Oct 2003, pp 29–43.
[^6]:What is Hadoop? - SAS. https://www.sas.com/zh_cn/insights/big-data/hadoop.html
[^7]:W. Tantisiriroj, S. Patil, G. Gibson. “Data-intensive file systems for Internet services: A rose by any other name ...” Technical Report CMUPDL-08-114, Parallel Data Laboratory, Carnegie Mellon University, Pittsburgh, PA, October 2008.