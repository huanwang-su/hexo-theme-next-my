---
title: RDD简介和RDD模型
date: 2018/5/12 15:30:25
category:
- 大数据
- spark
tag:
- spark
comments: true  
---

# RDD

弹性分布式数据集（Resilient Distributed Dataset，简称 RDD）。RDD 其实就是分布式的元素集合。在 Spark 中，对数据的所有操作主要是创 建 RDD、转化已有 RDD 以及调用 RDD 操作进行求值。

Spark 中的 RDD 就是一个不可变的分布式对象集合。每个 RDD 都被分为多个分区，这些 分区运行在集群中的不同节点上。

### 创建 RDD

Spark 提供了两种创建 RDD 的方式：

1. **读取一个外部数据集**

   `sc.textFile("README.md")`

2. **驱动器程序中对一个集合进 行并行化**

   `val in =sc.parallelize(List("pandas", "i like pandas"),3)`

### 操作RDD

RDD 支持两种类型的操作：**转化操作**（transformation）和**行动操作**（action）。转化操作会由一个 RDD 生成一个新的 RDD，行动操作会对 RDD 计算出一个结果，并把结果返回到Driver程序中，或把结果存储到外部存储系统（如 HDFS）中。

> 惰性计算，转化操作和行动操作的区别在于 Spark 计算 RDD 的方式不同。 Spark只有第一次在一个行动操作中用到 时，才会真正计算。

转化操作：`pythonLines = lines.filter(lambda line: "Python" in line)`

行动操作：`pythonLines.first(), pythonLines.persist() `

### Spark程序过程

总的来说，每个 Spark 程序或 shell 会话都按如下方式工作。

1. 从外部数据创建出输入 RDD。
2. 使用诸如 ﬁlter() 这样的转化操作对 RDD 进行转化，以定义新的 RDD。
3. 告诉 Spark 对需要被重用的中间结果 RDD 执行 persist() 操作。
4. 使用行动操作（例如 count() 和 ﬁrst() 等）来触发一次并行计算，Spark 会对计算进行优化后再执行。

## RDD特点

RDD是一种分布式的内存抽象，表示一个只读的记录分区的集合，它只能通过其他RDD转换而创建，RDD支持丰富的转换操作(如map, join, filter, groupBy等)，通过这种转换操作，新的RDD则包含了如何从其他RDDs衍生所必需的信息，所以说RDDs之间是有==依赖关系==的。基于RDDs之间的依赖，RDDs会形成一个==有向无环图DAG==，该DAG描述了整个流式计算的流程，实际执行的时候，RDD是通过==血缘关系(Lineage)==一气呵成的，即使出现数据分区丢失，也可以通过血缘关系重建分区，总结起来，基于RDD的流式计算任务可描述为：从稳定的物理存储(如分布式文件系统)中加载记录，记录被传入由一组确定性操作构成的DAG，然后写回稳定存储。另外RDD还可以将数据集缓存到内存中，使得在多个操作之间可以重用数据集，基于这个特点可以很方便地构建迭代型应用(图计算、机器学习等)或者交互式数据分析应用。

RDD表示只读的分区的数据集，对RDD进行改动，只能通过RDD的转换操作，由一个RDD得到一个新的RDD，新的RDD包含了从其他RDD衍生所必需的信息。RDDs之间存在依赖，RDD的执行是按照血缘关系延时计算的。如果血缘关系较长，可以通过==持久化RDD来切断血缘关系==。

### 分区

RDD以partition作为最小存储和计算单元，分布在cluster的不同nodes上，一个node可以有多个partitions，一个partition只能在一个node上

[![rdd-partition](http://sharkdtu.com/images/rdd-partition.png)](http://sharkdtu.com/images/rdd-partition.png)

### 并行

一个Task对应一个partition，Tasks之间相互独立可以并行计算

### 只读

如下图所示，RDD是只读的，要想改变RDD中的数据，只能在现有的RDD基础上创建新的RDD。

[![rdd-readonly](http://sharkdtu.com/images/rdd-readonly.png)](http://sharkdtu.com/images/rdd-readonly.png)

由一个RDD转换到另一个RDD，可以通过丰富的操作算子实现，不再像MapReduce那样只能写map和reduce了，如下图所示。

[![rdd-transform](http://sharkdtu.com/images/rdd-transform.png)](http://sharkdtu.com/images/rdd-transform.png)

RDD的操作算子包括两类，一类叫做transformations，它是用来将RDD进行转化，构建RDD的血缘关系；另一类叫做actions，它是用来触发RDD的计算，得到RDD的相关计算结果或者将RDD保存的文件系统中。下图是RDD所支持的操作算子列表。

[![rdd-transforms-actions](http://sharkdtu.com/images/rdd-transformations-actions.png)](http://sharkdtu.com/images/rdd-transformations-actions.png)

### 依赖

RDDs通过操作算子进行转换，转换得到的新RDD包含了从其他RDDs衍生所必需的信息，RDDs之间维护着这种血缘关系，也称之为依赖。

如下图所示，依赖包括两种

- 窄依赖（Narrow）是指父RDD的每个分区只被子RDD的一个分区所使用，**子RDD分区通常对应常数个父RDD分区**(O(1)，与数据规模无关)
- 宽依赖（shuffle）是指父RDD的每个分区都可能被**多个**子RDD分区所使用，**子RDD分区通常对应多个的父RDD分区**(O(n)，与数据规模有关)

宽依赖和窄依赖如下图所示：

![宽依赖和窄依赖示例](https://img-blog.csdn.net/20160913233559680)

通过RDDs之间的这种依赖关系，一个任务流可以描述为DAG(有向无环图)，如下图所示，在实际执行过程中宽依赖对应于Shuffle(图中的reduceByKey和join)，窄依赖中的所有转换操作可以通过类似于管道的方式一气呵成执行(图中map和union可以一起执行)。

[![rdd-dag](http://sharkdtu.com/images/rdd-dag.png)](http://sharkdtu.com/images/rdd-dag.png)

### 缓存

如果在应用程序中多次使用同一个RDD，可以将该RDD缓存起来，该RDD只有在第一次计算的时候会根据血缘关系得到分区的数据，在后续其他地方用到该RDD的时候，会直接从缓存处取而不用再根据血缘关系计算，这样就加速后期的重用。

#### checkpoint

虽然RDD的血缘关系天然地可以实现容错，当RDD的某个分区数据失败或丢失，可以通过血缘关系重建。但是对于长时间迭代型应用来说，随着迭代的进行，RDDs之间的血缘关系会越来越长，一旦在后续迭代过程中出错，则需要通过非常长的血缘关系去重建，势必影响性能。为此，RDD支持checkpoint将数据保存到持久化的存储中，这样就可以切断之前的血缘关系，因为checkpoint后的RDD不需要知道它的父RDDs了，它可以从checkpoint处拿到数据。

## RDD操作

RDDs support 两种类型的操作: *transformations（转换）*, 和 *actions（动作）*

Spark 可以从 Hadoop 所支持的任何存储源中创建 distributed dataset（分布式数据集），包括本地文件系统，HDFS，Cassandra，HBase，[Amazon S3](http://wiki.apache.org/hadoop/AmazonS3) 等等。 Spark 支持文本文件，[SequenceFiles](http://hadoop.apache.org/common/docs/current/api/org/apache/hadoop/mapred/SequenceFileInputFormat.html)，以及任何其它的 Hadoop [InputFormat](http://hadoop.apache.org/docs/stable/api/org/apache/hadoop/mapred/InputFormat.html)。

可以使用 `SparkContext` 的 `textFile` 方法来创建文本文件的 RDD。此方法需要一个文件的 URI（计算机上的本地路径 ，`hdfs://`，`s3n://` 等等的 URI），并且读取它们作为一个 lines（行）的集合。

### 传递 Functions（函数）给 Spark

当 driver 程序在集群上运行时，Spark 的 API 在很大程度上依赖于传递函数。有 2 种推荐的方式:

- [Anonymous function syntax（匿名函数语法）](http://docs.scala-lang.org/tutorials/tour/anonymous-function-syntax.html), 它可以用于短的代码片断.
- 在全局单例对象中的静态方法. 例如, 您可以定义 `object MyFunctions` 然后传递 `MyFunctions.func1`, 如下:

```
object MyFunctions {
  def func1(s: String): String = { ... }
}

myRdd.map(MyFunctions.func1)
```

请注意，虽然也有可能传递一个类的实例（与单例对象相反）的方法的引用，这需要发送整个对象，包括类中其它方法。例如，考虑:

```
class MyClass {
  def func1(s: String): String = { ... }
  def doStuff(rdd: RDD[String]): RDD[String] = { rdd.map(func1) }
}
```

这里，如果我们创建一个 `MyClass` 的实例，并调用 `doStuff`，在 `map` 内有 `MyClass` 实例的 `func1` 方法的引用，所以整个对象需要被发送到集群的。它类似于 `rdd.map(x => this.func1(x))`

类似的方式，访问外部对象的字段将引用整个对象:

```
class MyClass {
  val field = "Hello"
  def doStuff(rdd: RDD[String]): RDD[String] = { rdd.map(x => field + x) }
}
```

相当于写 `rdd.map(x => this.field + x)`, 它引用 `this` 所有的东西. 为了避免这个问题, 最简单的方式是复制 `field` 到一个本地变量，而不是外部访问它:

```
def doStuff(rdd: RDD[String]): RDD[String] = {
  val field_ = this.field
  rdd.map(x => field_ + x)
}
```

### 理解闭包 

一个关于 Spark 更难的事情是理解变量和方法的范围和生命周期. 修改其范围之外的变量 RDD 操作可以混淆的常见原因。在下面的例子中，我们将看一下使用的 `foreach()` 代码递增累加计数器，但类似的问题，也可能会出现其他操作上.

#### 示例

考虑一个简单的 RDD 元素求和，以下行为可能不同，具体取决于是否在同一个 JVM 中执行. 一个常见的例子是当 Spark 运行在 `local` 本地模式（`--master = local[n]`）时，与部署 Spark 应用到群集（例如，通过 spark-submit 到 YARN）:

```
var counter = 0
var rdd = sc.parallelize(data)
rdd.foreach(x => counter += x)
println("Counter value: " + counter)
```

#### Local（本地）vs. cluster（集群）模式

上面的代码行为是不确定的，并且可能无法按预期正常工作。执行作业时，Spark 会分解 RDD 操作到每个 executor 中的 task 里。在执行之前，Spark 计算任务的 **closure**（闭包）。闭包是指 executor 要在RDD上进行计算时必须对执行节点可见的那些变量和方法（在这里是foreach()）。闭包被序列化并被发送到每个 executor。

闭包的变量副本发给每个 **executor** ，==当 **counter** 被 `foreach` 函数引用的时候，它已经不再是 driver node 的 **counter** 了==。虽然在 driver node 仍然有一个 counter 在内存中，但是对 executors 已经不可见。executor 看到的只是序列化的闭包一个副本。所以 **counter** 最终的值还是 0，因为对 `counter` 所有的操作均引用序列化的 closure 内的值。

在 `local` 本地模式，在某些情况下的 `foreach` 功能实际上是同一 JVM 上的驱动程序中执行，并会引用同一个原始的 **counter** 计数器，实际上可能更新.

为了确保这些类型的场景明确的行为应该使用的 [`Accumulator`](http://spark.apachecn.org/docs/cn/2.2.0/rdd-programming-guide.html#accumulators) 累加器。当一个执行的任务分配到集群中的各个 worker 结点时，Spark 的累加器是专门提供安全更新变量的机制。

在一般情况下，closures - constructs 像循环或本地定义的方法，==不应该被用于改动一些全局状态==。Spark 没有规定或保证突变的行为，以从封闭件的外侧引用的对象。一些代码，这可能以本地模式运行，但是这只是偶然和这样的代码如预期在分布式模式下不会表现。如果需要一些全局的聚合功能，应使用 Accumulator（累加器）。

#### 打印 RDD 的 elements

另一种常见的语法用于打印 RDD 的所有元素使用 `rdd.foreach(println)` 或 `rdd.map(println)`。在一台机器上，这将产生预期的输出和打印 RDD 的所有元素。然而，在集群 `cluster` 模式下，`stdout` 输出正在被执行写操作 executors 的 `stdout` 代替，而不是在一个驱动程序上，==因此`stdout` 的 `driver` 程序不会显示这些==！==要打印 `driver` 程序的所有元素，可以使用的 `collect()` 方法首先把 RDD 放到 driver 程序节点上: `rdd.collect().foreach(println)`。这可能会导致 driver 程序耗尽内存==，虽说，因为 `collect()` 获取整个 RDD 到一台机器; 如果你只需要打印 RDD 的几个元素，一个更安全的方法是使用 `take()`: `rdd.take(100).foreach(println)`。

### Transformations（转换）

- Value数据类型的Transformation算子这种变换并不触发提交作业针对处理的数据项是Value型的数据。


- Key-Value数据类型的Transfromation算子这种变换并不触发提交作业针对处理的数据项是Key-Value型的数据对。

#### Value型

处理数据类型为Value型的Transformation算子可以根据RDD变换算子的输入分区与输出分区关系分为以下几种类型:

> 1）输入分区与输出分区一对一型 
> 2）输入分区与输出分区多对一型 
> 3）输入分区与输出分区多对多型 
> 4）输出分区为输入分区子集型 
> 5）还有一种特殊的输入与输出分区一对一的算子类型：Cache型。 Cache算子对RDD分区进行缓存

##### 输入分区与输出分区一对一型

- **map**：数据集中的每个元素经过用户自定义的函数转换形成一个新的RDD，新的RDD叫MappedRDD.

  ![img](http://images2015.cnblogs.com/blog/855959/201607/855959-20160731201803075-1566603859.png) 

- **flatMap**，与map类似，但每个元素输入项都可以被映射到0个或多个的输出项，最终将结果”扁平化“后输出。

  ![img](http://images2015.cnblogs.com/blog/855959/201607/855959-20160731201828294-509624797.png) 

- **mapPartitions**，mapPartitions函数获取到每个分区的迭代器，在函数中通过这个分区整体的迭代器对整个分区的元素进行操作。 内部实现是生成MapPartitionsRDD。

  类似与map，map作用于每个分区的每个元素，但mapPartitions作用于每个分区中，调用参数不同

  func的类型：Iterator[T] => Iterator[U]

  下图中用户通过函数 f (iter)=>iter.f ilter(_>=3) 对分区中所有数据进行过滤大于和等于 3 的数据保留。

  ![img](http://images2015.cnblogs.com/blog/855959/201607/855959-20160731202140622-1302550096.png) 

- **mapPartitionsWithIndex**

  与mapPartitions类似，不同的是函数多了个分区索引的参数

  func类型：(Int, Iterator[T]) => Iterator[U]

- **glom**

  glom函数将每个分区形成一个数组，内部实现是返回的RDD[Array[T]]。

  ![img](http://images2015.cnblogs.com/blog/855959/201607/855959-20160731220004841-310104312.png) 

##### 输入分区与输出分区多对一型

- **union**，使用union函数时需要保证两个RDD元素的数据类型相同，返回的RDD数据类型和被合并的RDD元素数据类型相同，并不进行去重操作，保存所有元素。如果想去重，可以使用distinct（）。++符号相当于uion函数操作。

  ![img](http://images2015.cnblogs.com/blog/855959/201607/855959-20160731202333106-98840694.png) 

- **cartesian**，笛 卡 尔 积 操 作。 操 作 后 内 部 实 现 返 回CartesianRDD

  ```scala
  // 参数是另外一个rdd，所以是一对多
  def cartesian[U: ClassTag](other: RDD[U]): RDD[(T, U)] = withScope {
    new CartesianRDD(sc, this, other)
  }
  ```

  ![img](http://images2015.cnblogs.com/blog/855959/201607/855959-20160731202551341-260166576.png) 

##### 输入分区与输出分区多对多型

- **groupBy**，将元素通过函数生成相应的Key，数据就转化为Key-Value格式，之后将Key相同的元素分为一组。

  ```
  def groupBy[K](f: T => K)(implicit kt: ClassTag[K]): RDD[(K, Iterable[T])]
  ```
  ![img](http://images2015.cnblogs.com/blog/855959/201607/855959-20160731203150309-1392947847.png) 

##### 输出分区为输入分区子集型

- **filter**，filter的功能是对元素进行过滤，对每个元素应用f函数，返回值为true的元素在RDD中保留，返回为false的将过滤掉。 内部实现相当于生成`FilteredRDD(this，sc.clean(f))。`

  ![img](http://images2015.cnblogs.com/blog/855959/201607/855959-20160731224355278-1057893706.png) 

- **distinct**，distinct将RDD中的元素进行去重操作。

  ![img](http://images2015.cnblogs.com/blog/855959/201607/855959-20160731220649481-586314519.png) 

- **subtract**，subtract相当于进行集合的差操作，RDD 1去除RDD 1和RDD 2交集中的所有元素。 

  ![img](http://images2015.cnblogs.com/blog/855959/201607/855959-20160731220749841-222338560.png) 

- **sample**，sample将RDD这个集合内的元素进行采样，获取所有元素的子集。用户可以设定是否有放回的抽样、百分比、随机种子，进而决定采样方式。 

  ```scala
  def sample(
      withReplacement: Boolean,
      fraction: Double,
      seed: Long = Utils.random.nextLong): RDD[T])
  ```

  ![img](http://images2015.cnblogs.com/blog/855959/201607/855959-20160731204116606-30482327.png) 

- **takeSample**，takeSample()函数和上面的sample函数是一个原理，但是不使用相对比例采样，而是按设定的采样个数进行采样，同时返回结果不再是RDD，而是相当于对采样后的数据进行collect()，返回结果的集合为单机的数组。

  ![img](http://images2015.cnblogs.com/blog/855959/201607/855959-20160731221057966-726438556.png) 

##### cache型

- **cache**，cache将RDD元素从磁盘缓存到内存，相当于persist（MEMORY_ONLY）函数的功能。 

- **persist**，persist函数对RDD进行缓存操作。数据缓存在哪里由StorageLevel枚举类型确定。有几种类型的组合，DISK代表磁盘，MEMORY代表内存，SER代表数据是否进行序列化存储。

  | Storage Level（存储级别）              | Meaning（含义）                                              |
  | -------------------------------------- | ------------------------------------------------------------ |
  | MEMORY_ONLY                            | 将 RDD 以反序列化的 Java 对象的形式存储在 JVM 中. 如果内存空间不够，部分数据分区将不再缓存，在每次需要用到这些数据时重新进行计算. 这是默认的级别. |
  | MEMORY_AND_DISK                        | 将 RDD 以反序列化的 Java 对象的形式存储在 JVM 中。如果内存空间不够，将未缓存的数据分区存储到磁盘，在需要使用这些分区时从磁盘读取. |
  | MEMORY_ONLY_SER (Java and Scala)       | 将 RDD 以序列化的 Java 对象的形式进行存储（每个分区为一个 byte 数组）。这种方式会比反序列化对象的方式节省很多空间，尤其是在使用 [fast serializer](http://spark.apachecn.org/docs/cn/2.2.0/tuning.html) 时会节省更多的空间，但是在读取时会增加 CPU 的计算负担. |
  | MEMORY_AND_DISK_SER (Java and Scala)   | 类似于 MEMORY_ONLY_SER ，但是溢出的分区会存储到磁盘，而不是在用到它们时重新计算. |
  | DISK_ONLY                              | 只在磁盘上缓存 RDD.                                          |
  | MEMORY_ONLY_2, MEMORY_AND_DISK_2, etc. | 与上面的级别功能相同，只不过每个分区在集群中两个节点上建立副本. |
  | OFF_HEAP (experimental 实验性)         | 类似于 MEMORY_ONLY_SER, 但是将数据存储在 [off-heap memory](http://spark.apachecn.org/docs/cn/2.2.0/configuration.html#memory-management) 中. 这需要启用 off-heap 内存. |

  > 在 shuffle 操作中（例如 `reduceByKey`），即便是用户没有调用 `persist` 方法，Spark 也会自动缓存部分中间数据.这么做的目的是，在 shuffle 的过程中某个节点运行失败时，不需要重新计算所有的输入数据。如果用户想多次使用某个 RDD，强烈推荐在该 RDD 上调用 persist 方法.
  >
  > **Spark 的存储级别的选择**，核心问题是在 memory 内存使用率和 CPU 效率之间进行权衡。建议按下面的过程进行存储级别的选择:
  >
  > - 如果您的 RDD 适合于默认存储级别 (`MEMORY_ONLY`), leave them that way. 这是CPU效率最高的选项，允许RDD上的操作尽可能快地运行.
  > - 如果不是, 试着使用 `MEMORY_ONLY_SER` 和 [selecting a fast serialization library](http://spark.apachecn.org/docs/cn/2.2.0/tuning.html) 以使对象更加节省空间，但仍然能够快速访问。 (Java和Scala)
  > - 不要溢出到磁盘，除非计算您的数据集的函数是昂贵的, 或者它们过滤大量的数据. 否则, 重新计算分区可能与从磁盘读取分区一样快.
  > - 如果需要快速故障恢复，请使用复制的存储级别 (e.g. 如果使用Spark来服务 来自网络应用程序的请求). *All* 存储级别通过重新计算丢失的数据来提供完整的容错能力，但复制的数据可让您继续在 RDD 上运行任务，而无需等待重新计算一个丢失的分区.

#### key-value型

##### 输入分区与输出分区一对一

- **mapValues**，mapValues：针对（Key，Value）型数据中的Value进行Map操作，而不对Key进行处理。 

  ```scala
    def mapValues[U](f: V => U): RDD[(K, U)] = {
      val cleanF = self.context.clean(f)
      new MapPartitionsRDD[(K, U), (K, V)](self,
        (context, pid, iter) => iter.map { case (k, v) => (k, cleanF(v)) },
        preservesPartitioning = true)
    }
  ```

##### 单个RDD或两个RDD聚集

- **combineByKey**, 对单个Rdd的聚合。相当于将元素为（Int，Int）的RDD转变为了（Int，Seq[Int]）类型元素的RDD。 
  定义combineByKey算子的说明如下：

  ```scala
  def combineByKey[C](
      createCombiner: V => C, // 在C不存在的情况下，如通过V创建seq C。
      mergeValue: (C, V) => C, // 当C已经存在的情况下，需要merge，如把item V加到seq 
      mergeCombiners: (C, C) => C, // 合并两个C
      partitioner: Partitioner, // Partitioner（分区器），Shuffle时需要通过Partitioner的分区策略进行分区。
      mapSideCombine: Boolean = true, // 为了减小传输量，很多combine可以在map端先做。例如， 叠加可以先在一个partition中把所有相同的Key的Value叠加， 再shuffle。
      serializer: Serializer = null // 传输需要序列化，用户可以自定义序列化类。
  	): RDD[(K, C)] 
  ```

  源码：

  ``` scala
    def combineByKey[C](createCombiner: V => C,
        mergeValue: (C, V) => C,
        mergeCombiners: (C, C) => C,
        partitioner: Partitioner,
        mapSideCombine: Boolean = true,
        serializer: Serializer = null): RDD[(K, C)] = {
      require(mergeCombiners != null, "mergeCombiners must be defined") // required as of Spark 0.9.0
      if (keyClass.isArray) {
        if (mapSideCombine) {
          throw new SparkException("Cannot use map-side combining with array keys.")
        }
        if (partitioner.isInstanceOf[HashPartitioner]) {
          throw new SparkException("Default partitioner cannot partition array keys.")
        }
      }
      val aggregator = new Aggregator[K, V, C](
        self.context.clean(createCombiner),
        self.context.clean(mergeValue),
        self.context.clean(mergeCombiners))
      if (self.partitioner == Some(partitioner)) {
        self.mapPartitions(iter => {
          val context = TaskContext.get()
          new InterruptibleIterator(context, aggregator.combineValuesByKey(iter, context))
        }, preservesPartitioning = true)
      } else {
        new ShuffledRDD[K, V, C](self, partitioner)
          .setSerializer(serializer)
          .setAggregator(aggregator)
          .setMapSideCombine(mapSideCombine)
      }
    }

    /**
     * Simplified version of combineByKey that hash-partitions the output RDD.
     */
    def combineByKey[C](createCombiner: V => C,
        mergeValue: (C, V) => C,
        mergeCombiners: (C, C) => C,
        numPartitions: Int): RDD[(K, C)] = {
      combineByKey(createCombiner, mergeValue, mergeCombiners, new HashPartitioner(numPartitions))
    }
  ```

- **reduceByKey**，reduceByKey是更简单的一种情况，只是两个值合并成一个值，所以createCombiner很简单，就是直接返回v，而mergeValue和mergeCombiners的逻辑相同，没有区别。 

  ``` scala
  def reduceByKey(partitioner: Partitioner, func: (V, V) => V): RDD[(K, V)] = {
      combineByKey[V]((v: V) => v, func, func, partitioner)
  }
  def reduceByKey(func: (V, V) => V, numPartitions: Int): RDD[(K, V)] = {
      reduceByKey(new HashPartitioner(numPartitions), func)
  }	
  ```

- **partitionBy**，partitionBy函数对RDD进行分区操作。 如果原有RDD的分区器和现有分区器（partitioner）一致，则不重分区，如果不一致，则相当于根据分区器生成一个新的ShuffledRDD。 

- **cogroup**，cogroup函数将两个RDD进行协同划分。对在两个RDD中的Key-Value类型的元素，每个RDD相同Key的元素分别聚合为一个集合，并且返回两个RDD中对应Key的元素集合的迭代器`(K, (Iterable[V], Iterable[w]))`。其中，Key和Value，Value是两个RDD下相同Key的两个数据集合的迭代器所构成的元组。 

  ![img](http://images2015.cnblogs.com/blog/855959/201607/855959-20160731221840606-1498363270.png) 

##### join

- **join**，join对两个需要连接的RDD进行cogroup函数操作。cogroup操作之后形成的新RDD，对每个key下的元素进行笛卡尔积操作，返回的结果再展平，对应Key下的所有元组形成一个集合，最后返回RDD[(K，(V，W))]。 
  join的本质是通过cogroup算子先进行协同划分，再通过flatMapValues将合并的数据打散。 

  ![img](http://images2015.cnblogs.com/blog/855959/201607/855959-20160731205235169-1186414997.png) 

- **leftOuterJoin**和**rightOuterJoin**，LeftOuterJoin（左外连接）和RightOuterJoin（右外连接）相当于在join的基础上先判断一侧的RDD元素是否为空，如果为空，则填充为空。 如果不为空，则将数据进行连接运算，并返回结果。

  ``` scala
  def leftOuterJoin[W](other: RDD[(K, W)], partitioner: Partitioner): RDD[(K, (V, Option[W]))] = {
      this.cogroup(other, partitioner).flatMapValues { pair =>
        if (pair._2.isEmpty) {
          pair._1.iterator.map(v => (v, None))
        } else {
          for (v <- pair._1.iterator; w <- pair._2.iterator) yield (v, Some(w))
        }
      }
    }

  ```

### Action算子

**本质上在Actions算子中通过SparkContext执行提交作业的runJob操作，触发了RDD DAG的执行。** 

根据Action算子的输出空间将Action算子进行分类：无输出、 HDFS、 Scala集合和数据类型。

#### 无输出

- **foreach**

  对RDD中的每个元素都应用f函数操作，不返回RDD和Array，而是返回Uint。 

  ``` scala
  def foreach(f: T => Unit) {
      val cleanF = sc.clean(f)
      sc.runJob(this, (iter: Iterator[T]) => iter.foreach(cleanF))
    }
  ```

#### HDFS

- **saveAsTextFile**, 函数将数据输出，存储到HDFS的指定目录。将RDD中的每个元素映射转变为（Null，x.toString），然后再将其写入HDFS。 
- **saveAsObjectFile**, saveAsObjectFile将分区中的每10个元素组成一个Array，然后将这个Array序列化，映射为（Null，BytesWritable（Y））的元素，写入HDFS为SequenceFile的格式。 

#### Scala集合和数据类型

- **collect**, collect将分布式的RDD返回为一个单机的scala Array数组。 在这个数组上运用scala的函数式操作。 

  ``` scala
    /**
     * Return an array that contains all of the elements in this RDD.
     */
    def collect(): Array[T] = {
      val results = sc.runJob(this, (iter: Iterator[T]) => iter.toArray)
      Array.concat(results: _*)
    }
  ```

- **collectAsMap**, collectAsMap对（K，V）型的RDD数据返回一个单机HashMap。对于重复K的RDD元素，后面的元素覆盖前面的元素。

  ``` scala
    def collectAsMap(): Map[K, V] = {
      val data = self.collect()
      val map = new mutable.HashMap[K, V]
      map.sizeHint(data.length)
      data.foreach { pair => map.put(pair._1, pair._2) }
      map
    }
  ```

- **reduceByKeyLocally**, 实现的是先reduce再collectAsMap的功能，先对RDD的整体进行reduce操作，然后再收集所有结果返回为一个HashMap。

- **lookup**, Lookup函数对（Key，Value）型的RDD操作，返回指定Key对应的元素形成的Seq。这个函数处理优化的部分在于，如果这个RDD包含分区器，则只会对应处理K所在的分区，然后返回由（K，V）形成的Seq。如果RDD不包含分区器，则需要对全RDD元素进行暴力扫描处理，搜索指定K对应的元素。 

- **count**, count返回整个RDD的元素个数。 

- **top**, top可返回最大的k个元素。

  相近函数说明：

  - top返回最大的k个元素。
  - take返回最小的k个元素。
  - takeOrdered返回最小的k个元素， 并且在返回的数组中保持元素的顺序。
  - first相当于top（ 1） 返回整个RDD中的前k个元素， 可以定义排序的方式Ordering[T]。返回的是一个含前k个元素的数组。

- **reduce**, reduce函数相当于对RDD中的元素进行reduceLeft函数的操作。reduceLeft先对两个元素进行reduce函数操作然后将结果和迭代器取出的下一个元素进行reduce函数操作直到迭代器遍历完所有元素得到最后结果。在RDD中先对每个分区中的所有元素的集合分别进行reduceLeft。 每个分区形成的结果相当于一个元素再对这个结果集合进行reduceleft操作。==即先计算各个分区，最后再合并结果==

- **flod**， fold和reduce的原理相同但是与reduce不同相当于每个reduce时迭代器取的第一个元素是zeroValue。

  `def fold(zeroValue: T)(op: (T, T) => T)`

- **aggregate**， aggregate先对每个分区的所有元素进行aggregate操作再对分区的结果进行fold操作。
  **aggreagate与fold和reduce的不同之处在于aggregate相当于采用归并的方式进行数据聚集这种聚集是并行化的。 而在fold和reduce函数的运算过程中每个分区中需要进行串行处理每个分区串行计算完结果结果再按之前的方式进行聚集并返回最终聚集结果。**

## 共享变量

通常情况下，一个传递给 Spark 操作（例如 `map` 或 `reduce`）的函数 func 是在远程的集群节点上执行的。该函数 func 在多个节点执行过程中使用的变量，是同一个变量的多个副本。这些变量的以副本的方式拷贝到每个机器上，并且各个远程机器上变量的更新并不会传播回 driver program（驱动程序）。Spark 提供了两种特定类型的共享变量 : broadcast variables（广播变量）和 accumulators（累加器）。

### 广播变量

Broadcast variables（广播变量）允许程序员将一个 ==read-only（只读的）==变量缓存到每台机器上，而不是给任务传递一个副本。它们是如何来使用呢，例如，广播变量可以用一种高效的方式给每个节点传递一份比较大的 input dataset（输入数据集）副本。在使用广播变量时，Spark 也尝试使用高效广播算法分发 broadcast variables（广播变量）以降低通信成本。

Spark 的 action（动作）操作是通过一系列的 stage（阶段）进行执行的，这些 stage（阶段）是通过分布式的 “shuffle” 操作进行拆分的。Spark 会自动广播出每个 stage（阶段）内任务所需要的公共数据。这种情况下广播的数据使用序列化的形式进行缓存，并在每个任务运行前进行反序列化。这也就意味着，只有在跨越多个 stage（阶段）的多个任务会使用相同的数据，或者在使用反序列化形式的数据特别重要的情况下，使用广播变量会有比较好的效果。

广播变量通过在一个变量 `v` 上调用 `SparkContext.broadcast(v)` 方法来进行创建。广播变量是 `v` 的一个 wrapper（包装器），可以通过调用 `value`方法来访问它的值。代码示例如下:

```scala
scala> val broadcastVar = sc.broadcast(Array(1, 2, 3))
broadcastVar: org.apache.spark.broadcast.Broadcast[Array[Int]] = Broadcast(0)

scala> broadcastVar.value
res0: Array[Int] = Array(1, 2, 3)
```

在创建广播变量之后，在集群上执行的所有的函数中，应该使用该广播变量代替原来的 `v` 值，所以节点上的 `v` 最多分发一次。另外，对象 `v` 在广播后不应该再被修改，以保证分发到所有的节点上的广播变量具有同样的值（例如，如果以后该变量会被运到一个新的节点）。

### Accumulators（累加器）

Accumulators（累加器）是一个仅可以执行 “added”（添加）的变量来通过一个关联和交换操作，因此可以高效地执行支持并行。累加器可以用于实现 counter（ 计数，类似在 MapReduce 中那样）或者 sums（求和）。原生 Spark 支持数值型的累加器，并且程序员可以添加新的支持类型。

作为一个用户，您可以创建 accumulators（累加器）并且重命名. 如下图所示, 一个命名的 accumulator 累加器（在这个例子中是 `counter`）将显示在 web UI 中，用于修改该累加器的阶段。 Spark 在 “Tasks” 任务表中显示由任务修改的每个累加器的值.

![Accumulators in the Spark UI](http://spark.apachecn.org/docs/cn/2.2.0/img/spark-webui-accumulators.png)

在 UI 中跟踪累加器可以有助于了解运行阶段的进度（注: 这在 Python 中尚不支持）.

## 分区

RDD 内部，如何表示并行计算的一个计算单元。答案是使用分区（Partition）

RDD 内部的数据集合在逻辑上和物理上被划分成多个小子集合，这样的每一个子集合我们将其称为分区，分区的个数会决定并行计算的粒度，而每一个分区数值的计算都是在一个单独的任务中进行，因此并行任务的个数，也是由 RDD分区的个数决定的

``` scala
trait Partition extends Serializable {
  def index: Int
  override def hashCode(): Int = index
}
```

RDD 只是数据集的抽象，分区内部并不会存储具体的数据。`Partition` 类内包含一个 `index` 成员，表示该分区在 RDD 内的编号，通过 RDD 编号 + 分区编号可以唯一确定该分区对应的块编号，利用底层数据存储层提供的接口，就能从存储介质（如：HDFS、Memory）中提取出分区对应的数据。

### 分区个数

RDD 分区的一个分配原则是：尽可能使得分区的个数，等于集群核心数目。

## 分区器

### 数据倾斜

数据倾斜指的是，并行处理的数据集中，某一部分（如Spark或Kafka的一个Partition）的数据显著多于其它部分，从而使得该部分的处理速度成为整个数据集处理的瓶颈。

### 分区的作用

在PairRDD即（key,value）这种格式的rdd中，很多操作都是基于key的，因此为了独立分割任务，会按照key对数据进行重组。比如groupbykey

![img](https://images2015.cnblogs.com/blog/449064/201704/449064-20170416134041196-343215838.png) 

重组肯定是需要一个规则的，最常见的就是基于Hash，Spark还提供了一种稍微复杂点的基于抽样的Range分区方法。

下面我们先看看分区器在Spark计算流程中是怎么使用的：

### Paritioner的使用

``` scala
abstract class Partitioner extends Serializable {
  def numPartitions: Int
  def getPartition(key: Any): Int
}
```

就拿groupbykey来说：

```scala
def groupByKey(): JavaPairRDD[K, JIterable[V]] =
    fromRDD(groupByResultToJava(rdd.groupByKey()))
```

它会调用PairRDDFunction的groupByKey()方法

```scala
def groupByKey(): RDD[(K, Iterable[V])] = self.withScope {
    groupByKey(defaultPartitioner(self))
  }
```

在这个方法里面创建了默认的分区器。默认的分区器是这样定义的：

```scala
def defaultPartitioner(rdd: RDD[_], others: RDD[_]*): Partitioner = {
    val bySize = (Seq(rdd) ++ others).sortBy(_.partitions.size).reverse //获取最多分区数的RDD
    for (r <- bySize if r.partitioner.isDefined && r.partitioner.get.numPartitions > 0) { // 该RDD分区器定义有效
      return r.partitioner.get
    }
    // 返回hash RDD分区器
    if (rdd.context.conf.contains("spark.default.parallelism")) {	
      new HashPartitioner(rdd.context.defaultParallelism)
    } else {
      new HashPartitioner(bySize.head.partitions.size)
    }
  }
```

首先获取当前分区的分区个数，如果没有设置`spark.default.parallelism`参数，则创建一个跟之前分区个数一样的Hash分区器。

当然，用户也可以自定义分区器，或者使用其他提供的分区器。API里面也是支持的：

```scala
// 传入分区器对象
def groupByKey(partitioner: Partitioner): JavaPairRDD[K, JIterable[V]] =
    fromRDD(groupByResultToJava(rdd.groupByKey(partitioner)))
// 传入分区的个数
def groupByKey(numPartitions: Int): JavaPairRDD[K, JIterable[V]] =
    fromRDD(groupByResultToJava(rdd.groupByKey(numPartitions)))
```

### HashPatitioner

Hash分区器，是最简单也是默认提供的分区器

```scala
class HashPartitioner(partitions: Int) extends Partitioner {
  require(partitions >= 0, s"Number of partitions ($partitions) cannot be negative.")

  def numPartitions: Int = partitions

  // 通过key计算其HashCode，并根据分区数取模。如果结果小于0，直接加上分区数。
  def getPartition(key: Any): Int = key match {
    case null => 0
    case _ => Utils.nonNegativeMod(key.hashCode, numPartitions)
  }

  // 对比两个分区器是否相同，直接对比其分区个数就行
  override def equals(other: Any): Boolean = other match {
    case h: HashPartitioner =>
      h.numPartitions == numPartitions
    case _ =>
      false
  }

  override def hashCode: Int = numPartitions
}
```

这里最重要的是这个`Utils.nonNegativeMod(key.hashCode, numPartitions)`,它决定了数据进入到哪个分区。

```scala
def nonNegativeMod(x: Int, mod: Int): Int = {
    val rawMod = x % mod
    rawMod + (if (rawMod < 0) mod else 0)
  }
```

说白了，就是基于这个key获取它的hashCode，然后对分区个数取模。由于HashCode可能为负，这里直接判断下，如果小于0，再加上分区个数即可。

因此，基于hash的分区，只要保证你的key是分散的，那么最终数据就不会出现数据倾斜的情况。

### RangePartitioner

从HashPartitioner分区的实现原理我们可以看出，其结果可能导致每个分区中数据量的不均匀，极端情况下会导致某些分区拥有RDD的全部数据，这显然不是我们需要的。而RangePartitioner分区则尽量保证每个分区中数据量的均匀，而且分区与分区之间是有序的，也就是说一个分区中的元素肯定都是比另一个分区内的元素小或者大；==但是分区内的元素是不能保证顺序的。简单的说就是将一定范围内的数映射到某一个分区内。==

``` scala
class RangePartitioner[K: Ordering : ClassTag, V](
                                                   partitions: Int,
                                                   rdd: RDD[_ <: Product2[K, V]],
                                                   private var ascending: Boolean = true)
  extends Partitioner {

  // We allow partitions = 0, which happens when sorting an empty RDD under the default settings.
  require(partitions >= 0, s"Number of partitions cannot be negative but found $partitions.")

  // 获取RDD中key类型数据的排序器
  private var ordering = implicitly[Ordering[K]]

  // An array of upper bounds for the first (partitions - 1) partitions
  private var rangeBounds: Array[K] = {
    if (partitions <= 1) {
      // 如果给定的分区数是一个的情况下，直接返回一个空的集合，表示数据不进行分区
      Array.empty
    } else {
      // This is the sample size we need to have roughly balanced output partitions, capped at 1M.
      // 给定总的数据抽样大小，最多1M的数据量(10^6)，最少20倍的RDD分区数量，也就是每个RDD分区至少抽取20条数据
      val sampleSize = math.min(20.0 * partitions, 1e6)
      // Assume the input partitions are roughly balanced and over-sample a little bit.
      // 计算每个分区抽取的数据量大小， 假设输入数据每个分区分布的比较均匀
      // 对于超大数据集(分区数超过5万的)乘以3会让数据稍微增大一点，对于分区数低于5万的数据集，每个分区抽取数据量为60条也不算多
      val sampleSizePerPartition = math.ceil(3.0 * sampleSize / rdd.partitions.size).toInt
      // 从rdd中抽取数据，返回值:(总rdd数据量， Array[分区id，当前分区的数据量，当前分区抽取的数据])
      val (numItems, sketched) = RangePartitioner.sketch(rdd.map(_._1), sampleSizePerPartition)
      if (numItems == 0L) {
        // 如果总的数据量为0(RDD为空)，那么直接返回一个空的数组
        Array.empty
      } else {
        // If a partition contains much more than the average number of items, we re-sample from it
        // to ensure that enough items are collected from that partition.
        // 计算总样本数量和总记录数的占比，占比最大为1.0
        val fraction = math.min(sampleSize / math.max(numItems, 1L), 1.0)
        // 保存样本数据的集合buffer
        val candidates = ArrayBuffer.empty[(K, Float)]
        // 保存数据分布不均衡的分区id(数据量超过fraction比率的分区)
        val imbalancedPartitions = mutable.Set.empty[Int]
        // 计算抽取出来的样本数据
        sketched.foreach { case (idx, n, sample) =>
          if (fraction * n > sampleSizePerPartition) {
            // 如果fraction乘以当前分区中的数据量大于之前计算的每个分区的抽象数据大小，那么表示当前分区抽取的数据太少了，该分区数据分布不均衡，需要重新抽取
            imbalancedPartitions += idx
          } else {
            // 当前分区不属于数据分布不均衡的分区，计算占比权重，并添加到candidates集合中
            // The weight is 1 over the sampling probability.
            val weight = (n.toDouble / sample.size).toFloat
            for (key <- sample) {
              candidates += ((key, weight))
            }
          }
        }

        // 对于数据分布不均衡的RDD分区，重新进行数据抽样
        if (imbalancedPartitions.nonEmpty) {
          // Re-sample imbalanced partitions with the desired sampling probability.
          // 获取数据分布不均衡的RDD分区，并构成RDD
          val imbalanced = new PartitionPruningRDD(rdd.map(_._1), imbalancedPartitions.contains)
          // 随机种子
          val seed = byteswap32(-rdd.id - 1)
          // 利用rdd的sample抽样函数API进行数据抽样
          val reSampled = imbalanced.sample(withReplacement = false, fraction, seed).collect()
          val weight = (1.0 / fraction).toFloat
          candidates ++= reSampled.map(x => (x, weight))
        }

        // 将最终的抽样数据计算出rangeBounds出来
        RangePartitioner.determineBounds(candidates, partitions)
      }
    }
  }

  // 下一个RDD的分区数量是rangeBounds数组中元素数量+ 1个
  def numPartitions: Int = rangeBounds.length + 1

  // 二分查找器，内部使用java中的Arrays类提供的二分查找方法
  private var binarySearch: ((Array[K], K) => Int) = CollectionsUtils.makeBinarySearch[K]

  // 根据RDD的key值返回对应的分区id。从0开始
  def getPartition(key: Any): Int = {
    // 强制转换key类型为RDD中原本的数据类型
    val k = key.asInstanceOf[K]
    var partition = 0
    if (rangeBounds.length <= 128) {
      // If we have less than 128 partitions naive search
      // 如果分区数据小于等于128个，那么直接本地循环寻找当前k所属的分区下标
      while (partition < rangeBounds.length && ordering.gt(k, rangeBounds(partition))) {
        partition += 1
      }
    } else {
      // Determine which binary search method to use only once.
      // 如果分区数量大于128个，那么使用二分查找方法寻找对应k所属的下标;
      // 但是如果k在rangeBounds中没有出现，实质上返回的是一个负数(范围)或者是一个超过rangeBounds大小的数(最后一个分区，比所有数据都大)
      partition = binarySearch(rangeBounds, k)
      // binarySearch either returns the match location or -[insertion point]-1
      if (partition < 0) {
        partition = -partition - 1
      }
      if (partition > rangeBounds.length) {
        partition = rangeBounds.length
      }
    }

    // 根据数据排序是升序还是降序进行数据的排列，默认为升序
    if (ascending) {
      partition
    } else {
      rangeBounds.length - partition
    }
  }
```



#### **采样总数**

在新的rangeBounds算法总，采样总数做了一个限制，也就是最大只采样1e6的样本（也就是1000000）：

`val sampleSize =math.min(20.0 * partitions, 1e6)`

#### **父RDD中每个分区采样样本数**

　　按照我们的思路，正常情况下，父RDD每个分区需要采样的数据量应该是sampleSize/rdd.partitions.size，但是我们看代码的时候发现父RDD每个分区需要采样的数据量是正常数的3倍。

`val sampleSizePerPartition = math.ceil(3.0* sampleSize / rdd.partitions.size).toInt`

因为父RDD各分区中的数据量可能会出现倾斜的情况，乘于3的目的就是保证数据量小的分区能够采样到足够的数据，而对于数据量大的分区会进行第二次采样。

#### **采样算法**

　　这个地方就是RangePartitioner分区的核心了，其内部使用的就是水塘抽样，而这个抽样特别适合那种总数很大而且未知，并无法将所有的数据全部存放到主内存中的情况。也就是我们不需要事先知道RDD中元素的个数（不需要调用rdd.count()了！）。

> 水塘抽样
>
> 水塘抽样算法是用于解决，对于一个未知长度的数据流进行随机采样的问题的。
>
> 一个元素的时候：被抽中的概率为1
> 两个元素的时候：第一个元素留下的概率为1-1/2，第二个元素留下的概率为 1/2
> 三个元素的时候：第一个元素留下的概率为(1-1/2)*(1-1/3)=1/3，第二个元素留下的概率为1/2*(1-1/3)=1/3,第三个元素留下的概率为1/3
> 以此类推，当判断到第i个元素时，第m（0<m<=i）个元素留下的概率为$1/m*(1-1/(m+1))*(1-1/(m+2))...(1-1/(i))=1/i$就可以保证所有元素的概率相等

真正的抽样算法在SamplingUtils中,由于在Spark中是需要一次性取多个值的，因此直接去前n个数值，然后依次概率替换即可

最后就可以通过获取的样本数据，确定边界了。

```
// 从rdd中抽取数据，返回值:(总rdd数据量， Array[分区id，当前分区的数据量，当前分区抽取的数据])
      val (numItems, sketched) = RangePartitioner.sketch(rdd.map(_._1), sampleSizePerPartition)
```



![ img](https://images2015.cnblogs.com/blog/449064/201704/449064-20170416140016337-455449521.png) 

父RDD各分区中的数据量可能不均匀，在极端情况下，有些分区内的数据量会占有整个RDD的绝大多数的数据，如果按照水塘抽样进行采样，会导致该分区所采样的数据量不足，所以我们需要对该分区再一次进行采样，而这次采样使用的就是rdd的sample函数。

对于满足于`fraction * n > sampleSizePerPartition`条件的分区，我们对其再一次采样。所有采样完的数据全部存放在candidates 中。 

#### **确认边界**

从上面的采样算法可以看出，对于不同的分区weight的值是不一样的（不同分区的元素数量不一样，但每个分区采样数是一样的），这个值对应的就是每个分区的采样间隔。

RangePartitioner的determineBounds函数的作用是根据样本数据记忆权重大小确定数据边界, 代码注释讲解如下：

```  scala
def determineBounds[K: Ordering : ClassTag](
                                               candidates: ArrayBuffer[(K, Float)],
                                               partitions: Int): Array[K] = {
    val ordering = implicitly[Ordering[K]]
    // 按照数据进行数据排序，默认升序排列
    val ordered = candidates.sortBy(_._1)
    // 获取总的样本数量大小
    val numCandidates = ordered.size
    // 计算总的权重大小
    val sumWeights = ordered.map(_._2.toDouble).sum
    // 计算步长
    val step = sumWeights / partitions
    var cumWeight = 0.0
    var target = step
    val bounds = ArrayBuffer.empty[K]
    var i = 0
    var j = 0
    var previousBound = Option.empty[K]
    while ((i < numCandidates) && (j < partitions - 1)) {
      // 获取排序后的第i个数据及权重
      val (key, weight) = ordered(i)
      // 累计权重
      cumWeight += weight
      if (cumWeight >= target) {
        // Skip duplicate values.
        // 权重已经达到一个步长的范围，计算出一个分区id的值
        if (previousBound.isEmpty || ordering.gt(key, previousBound.get)) {
          // 上一个边界值为空，或者当前边界key数据大于上一个边界的值，那么当前key有效，进行计算
          // 添加当前key到边界集合中
          bounds += key
          // 累计target步长界限
          target += step
          // 分区数量加1
          j += 1
          // 上一个边界的值重置为当前边界的值
          previousBound = Some(key)
        }
      }
      i += 1
    }
    // 返回结果
    bounds.toArray
  }
```

### 自定义分区器

自定义分区器，也是很简单的，只需要实现对应的两个方法就行：

```scala
public class MyPartioner extends Partitioner {
    @Override
    public int numPartitions() {
        return 1000;
    }

    @Override
    public int getPartition(Object key) {
        String k = (String) key;
        int code = k.hashCode() % 1000;
        System.out.println(k+":"+code);
        return  code < 0?code+1000:code;
    }

    @Override
    public boolean equals(Object obj) {
        if(obj instanceof MyPartioner){
            if(this.numPartitions()==((MyPartioner) obj).numPartitions()){
                return true;
            }
            return false;
        }
        return super.equals(obj);
    }
}
```

使用的时候，可以直接new一个对象即可。

```
pairRdd.groupbykey(new MyPartitioner())
```

