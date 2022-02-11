+++
title = "翻译：RDD编程指导"
description = "http://spark.apache.org/docs/latest/rdd-programming-guide.html"
date = 2021-07-04T12:42:17+08:00

[taxonomies]
categories = ["Translations"]
tags = ["note", "spark", "apache", "big data"]

[extra]
toc = true
comments = true
+++

## 总览

每个spark应用包含一个驱动程序，两个职责:

1. 运行用户的main函数
2. 在集群上执行一些并行操作

{% mermaid() %}
graph TD;
  subgraph spark application;
  core[driver program] -.职责1.-> act1(run user's main function);
  core[driver program] -.职责2.-> act2(execute various parallel operations);
  end
{% end %}

同样，spark提供了两层抽象:

1. resilient distributed dataset(RDD): 一组元素的集合，分段地存储在集群不同节点上，并且支持并行操作。
	1. 可以是分布式文件系统中的一个文件
	2. 可以是driver program中的一个scala集合
3. shared variables: 可以在并行操作中使用。
	1. broadcast类型变量: 用于在不同节点地内存中存储一个值
	2. accumulators类型变量: 只支持"add"操作的变量

## 初始化spark

1. 初始化`SparkConf`，包含目标应用的信息
2. 使用`SparkConf`初始化`SparkContext`以连接cluster

## Resilient Distributed Datasets (RDDs)

### Parallelized Collections

调用`SparkContext`的`parallelize`方法，传入一个普通集合以生产一个并行集合，之后便可以并行操作。

此外，还可以控制并行集合的分区数量，spark会为并行集合的每一个分区运行一个任务。一般spark会自动设置合适的分区，当然也可以`parallelize`方法控制。

### External Datasets

spark可以从hadoop支持的任何数据源创建RDD，并且支持hadoop支持的所有`inputFormat`，包括text files、SequenceFiles。

spark读取文件(`textFile`方法)的一些提示:

1. 使用本地文件系统上的文件，，比如保证每个node相同路径上也有这个文件。
2. spark所有的文件读取的方法，支持目录、压缩文件以及通配符。
3. 可选的第二个参数用于控制分块数量。默认情况下，spark会根据文件block大小创建分块(hdfs下默认是128M)。

除了`textFile`外，spark还提供了其它格式的数据读取方法:

1. `SparkContext.wholeTextFiles`可以读取包含大量小文件的目录，并以文件名:文件内容对的方式返回。
2. 对于`SequenceFile`，使用`SparkContext.sequenceFile[K, V]`方法读取，K、V是文件中key和value的类型，并且应该是hadoop的`Writable`接口的实现者。
3. 对于hadoop的`inputFormat`，可以使用`SparkContext.hadoopRDD`方法。
4. `saveAsObjectFile`和`objectFile`方法用于保存RDD。

### RDD的操作

RDD支持两种类型的操作:

1. 转换(transformations): 从现有数据集中创建新的数据集
2. 计算(actions): 在某个数据集上运行并获取结果

举例，map操作就是转换，传入某个数据集的每个元素，返回代表着新数据集的RDD。reduce就是计算，聚合RDD中所有元素，返回最终结果到driver program。

**spark中所有的转换操作都是lazy的**，也就是说不会在调用时立马计算，而是在计算操作被执行时才去计算。

默认情况下，每次运行计算，所有转换过的RDD都会被重复计算。但是可以配置spark缓存。

#### 基础

举例说明(网站上例子):

```java
JavaRDD<String> lines = sc.textFile("data.txt");
JavaRDD<Integer> lineLengths = lines.map(s -> s.length());
int totalLength = lineLengths.reduce((a, b) -> a + b);
```

第一行定义了从文件中获取的RDD。注意，并未实际读取，而只是一个指针。

第二行对这个文件进行map转换操作。注意，因为懒加载，并未实际处理。

最后一行执行reduce计算操作。此时spark将计算任务分解为一个个task，运行在集群不同的节点上。每个节点在其part上运行map、reduce操作，并返回结果给driver program。


通过在reduce前调用
```java
lineLengths.persist(StorageLevel.MEMORY_ONLY());
```
可以将`lineLengths`RDD缓存起来，之后重复调用reduce时不必重复map。

#### 函数传递

spark的api严重依赖于**传递函数**的方式执行任务。具体请看官方网站。

#### 理解闭包

spark最难理解的是在群集中执行代码时*变量的范围*、*生命周期*和*方法*。

最容易犯的错就是在变量的作用域外调用RDD的operations。

##### 例子

```java
int counter = 0;
JavaRDD<Integer> rdd = sc.parallelize(data);

// Wrong: Don't do this!!
rdd.foreach(x -> counter += x);

println("Counter value: " + counter);
```

这段代码的结果取决于是否运行在同一个JVM中。也就是说，是本地运行 (--master = local[n]) 还是集群运行（spark-submit）。

运行jobs时，spark会将RDD的operations分解为tasks，交由一个个executor执行。而在执行前，spark会构建task的闭包。闭包是对RDD执行计算时必须要用到的变量(例子中就是counter)和方法(例子中就是foreach())，会被序列化并发送给各个executor。

在集群模式，闭包使用到的变量会被拷贝进闭包，然后发送给executor执行，executo后续会对闭包内的counter进行处理，而不是driver program中的counter。也就是说，修改并不成功，最终结果还是0。而local模式则相反，task运行在相同JVM，counter的引用就是原变量，因此可以得到正确结果。

在这个场景中，为了得到正确结果，spark定义了accumulator类型变量，提供了跨节点累加值的机制。

##### 输出RDD元素

RDD的另一种常见错误用法是通过forEach(println)和map(println)输出元素。在单例模式下正常运作，但是在集群模式下，打印的值会输出到executor的标准输出，而不是driver program的，因此也就看不到元素内容。

为了正确获取元素内容，需要先调用`collect()`函数以将元素收集到driver program，然后再调用输出函数。但是这容易造成OOM。另一种方案是`take()`函数，只收集一定数量的元素。

#### Key-Value对

虽然大多数Spark操作都适用于包含任何类型对象的RDDs，但有几个特殊的操作只适用于键值对的RDDs。最常见的是分布式 "洗牌(shuffle)"操作，如按键分组或聚合元素。

#### Transformations

请到网页查看: [transformations](http://spark.apache.org/docs/latest/rdd-programming-guide.html#transformations)

#### Actions

请到网页查看: [actions](http://spark.apache.org/docs/latest/rdd-programming-guide.html#actions)

#### 洗牌操作

Spark中的某些操作会触发一个被称为洗牌的事件。洗牌是Spark重新分配数据的机制，使其在不同的分区中以不同的方式分组。这通常涉及跨执行器和机器的数据复制，使洗牌成为一个复杂而昂贵的操作。

##### 背景

为了解释洗牌操作，用`reduceByKey`来演示，这个方法生成新的RDD，将所有于key关联的value合并为一个touple。最麻烦的地方是，不是所有和key关联的value在同一个分区、甚至节点。不同executor需要合作以得到结果。

在spark中，不会为了某个operation去交换分区。在计算过程中，一个task就只操作一个分区；因此，为了规划`reduceByKey`这个reduce任务，spark需要一个**all-to-all**操作: 从所有分区读取所有key对应的所有的的value，然后跨集群聚集所有value以计算每个key对应的结果集。这就是"洗牌"。

洗牌后排序看网站

##### 性能损失

洗牌操作非常消耗资源，需要磁盘I/O、序列化和网络I/O。为了聚集洗牌需要的数据，spark会启动一系列task，"map任务"用于组织数据，"reduce任务"用于聚合数据。

在spark内部，洗牌操作执行时，每个"map任务"会将结果保存在内存中(直到溢出，溢出则分块写入外存)，然后根据分区顺序排序后写入单一的文件内; "reduce任务"会读取这个文件。

洗牌还会产生大量临时文件，直到RDD生命周期结束并且相关资源被垃圾回收。注意，垃圾回收会在任务停止之后很久很久才进行，也就是说，长期的的spark job会导致大量存储消耗。

### 持久存储

RDD可以被持久化存储，每个节点都会将它计算的分区存储在内存中，并在该数据集（或由其衍生的数据集）上的其他操作中重复使用。

使用方法`persist()`或者`cache()`使RDD持久化。

除此之外，持久化存储级别是可以选择的，默认是内存。具体请看[网站](https://spark.apache.org/docs/latest/rdd-programming-guide.html#rdd-persistence)。

#### 选择持久化存储级别

持久化存储关系到内存和CPU的效率，官方给出选择方法，具体请看[网站](https://spark.apache.org/docs/latest/rdd-programming-guide.html#which-storage-level-to-choose)。

#### 移除数据

Spark监控缓存的使用，并依照LRU算法丢弃旧缓存。可以通过`unpersist()`方法取消缓存，默认异步完成；可以通过传入`blocking=true`控制为同步方式。

## 共享变量

Spark操作的运行类似与函数调用，值传递。为了实现在节点运行的操作改变driver program的数值，spark提供了两种变量类型：broadcast variable、accumulator。

### Broadcast Variables

广播变量可以让每个节点保存一个只读变量在本地，而不是每次运行task传入。用于高效地给每个节点一些较大数据集。

### Accumulators

累加器是只通过关联和换元操作 "只增"的变量，因此可以有效地支持并行，可以用来实现计数器（如MapReduce）或总和。Spark原生支持数字类型的累加器，程序员可以增加对新类型的支持。
