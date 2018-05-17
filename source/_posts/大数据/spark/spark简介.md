---
title: spark简介
date: 2018/5/12 15:30:25
category:
- 大数据
- spark
tag:
- spark
comments: true  
---

# spark简介

Spark 是一个用来实现快速而通用的==集群计算的平台==，Spark简单说就是内存计算（包含迭代式计算，DAG计算,流式计算 ）框架

spark的优点有：

1. **轻量级快速处理**。Spark允许Hadoop集群中的应用程序在内存中以100倍的速度运行，即使在磁盘上运行也能快10倍。Spark通过减少磁盘IO来达到性能提升，它们将中间处理数据全部放到了内存中。
2. **易于使用，Spark支持多语言。**Spark允许Java、Scala及Python，自带了80多个高等级操作符，允许在shell中进行交互式查询。
3. **支持复杂查询。**在简单的“map”及“reduce”操作之外，Spark还支持SQL查询、流式查询及复杂查询，比如开箱即用的机器学习机图算法。
4. **实时的流处理。**
5. **可以与Hadoop和已存Hadoop数据整合。**
6. **活跃和无限壮大的社区。**



## spark生态和组件

![这里写图片描述](https://img-blog.csdn.net/20171023112923566?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvQ19GdUw=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

- Spark Core: 包含spark的主要基本功能包括 任务调度、内存管理、错误恢复、与存储系统交互等模块。Spark Core 中还包含了对弹性分布式数据集（resilient distributed dataset，简称RDD）
- Spark SQL: Spark中用于结构化数据处理的软件包. 用户可以在Spark环境下用SQL语言处理数据. Spark SQL 支持多种数据源，比 如 Hive 表、Parquet 以及 JSON 等。
- Spark Streaming: Spark中用来处理流数据的部件 
- MLlib: Spark 中用来进行机器学习和数学建模的软件包 
- GraphX: Spark 中用来进行图计算的函数库 
- 集群管理器 ，Spark 支持在各种集群管理器（cluster manager）上运行，包括 Hadoop YARN、Apache Mesos，以及 Spark 自带的一个简易调度 器，叫作独立调度器。

## Spark基本概念

1. RDD——Resillient Distributed Dataset A Fault-Tolerant Abstraction for In-Memory Cluster Computing弹性分布式数据集。
2. Operation——作用于RDD的各种操作分为transformation和action。
   - **Transformations**：转换(Transformations) (如：map, filter, groupBy, join等)，Transformations操作是Lazy的，也就是说从一个RDD转换生成另一个RDD的操作不是马上执行，Spark在遇到Transformations操作时只会记录需要这样的操作，并不会去执行，需要等到有Actions操作的时候才会真正启动计算过程进行计算。
   - **Actions**：操作(Actions) (如：count, collect, save等)，Actions操作会返回结果或把RDD数据写到存储系统中。Actions是触发Spark启动计算的动因。
3. Job——作业，一个JOB包含多个RDD及作用于相应RDD上的各种operation。
4. Stage——一个作业分为多个阶段。
5. Partition——数据分区， 一个RDD中的数据可以分成多个不同的区。
6. DAG——Directed Acycle graph，有向无环图，反应RDD之间的依赖关系。
7. Narrow dependency——窄依赖，子RDD依赖于父RDD中固定的data partition。
8. Wide Dependency——宽依赖，子RDD对父RDD中的所有data partition都有依赖。
9. Caching Managenment——缓存管理，对RDD的中间计算结果进行缓存管理以加快整 体的处理速度。

## spark架构

Spark架构采用了分布式计算中的Master-Slave模型。Master是对应集群中的含有Master进程的节点，Slave是集群中含有Worker进程的节点。Master作为整个集群的控制器，负责整个集群的正常运行；Worker相当于是计算节点，接收主节点命令与进行状态汇报；Executor负责任务的执行；client作为用户的客户端负责提交应用，Driver负责控制一个应用的执行。具体如下图

![点击查看原始大小图片](http://dl2.iteye.com/upload/attachment/0109/5772/08f9d15b-10ea-3486-b6c9-a65809e27b0b.png)

在一个Spark应用的执行过程中，Driver和Worker是两个重要角色。Driver程序是应用逻辑执行的起点，负责作业的调度，即Task任务的分发，而多个Worker用来管理计算节点和创建Executor并行处理任务。在执行阶段，Driver会将Task和Task所依赖的file和jar序列化后传递给对应的Worker机器，同时Exucutor对相应数据分区的任务进行处理。

Spark的架构中的基本组件：

- ClusterManager：在Standalone模式中即为Master（主节点），控制整个集群，监控Worker。在YARN模式中为资源管理器。
- Worker：从节点，负责控制计算节点，启动Executor或Driver。在YARN模式中为NodeManager，负责计算节点的控制。
- Driver :运行Application的main（）函数并创建SparkContext
- Executor：执行器，在worker node上执行任务的组件、用于启动线程池运行任务。每个Application拥有独立的一组Executors。
- SparkContext：整个应用的上下文，控制应用的生命周期。
- RDD:Spark的基本计算单元，一组RDD可形成执行的有向无环RDD Graph
- DAG Scheduler:根据作业(Job)构建基于Stage的DAG,并提交Stage给TaskScheduler
- TaskScheduler：将任务（Task）分发给Executor执行
- SparkEnv：线程级别的上下文，存储运行时的重要组件的引用。 SparkEnv内创建并包含如下的一些重要组件的引用。
- MapOutPutTracker：负责Shuffle元信息的存储。
- BroadcastManager：负责广播变量的控制与元信息的存储。
- BlockManager：负责储存管理、创建和查找块。
- MetricsSystem：监控运行时性能指标信息。
- SparkConf：负责存储配置信息。

## spark运行逻辑

![点击查看原始大小图片](http://dl2.iteye.com/upload/attachment/0109/5774/5fce0961-1030-3805-b649-acafae85170b.png)

在Spark应用中，整个执行流程在逻辑上会形成有向无环图（DAG）。Action算子触发之后，将所有累积的算子形成一个有向无环图，然后由调度器调度该图上的任务进行运算。Spark的调度方式与MapReduce有所不同。Spark根据RDD之间不同的依赖关系切分形成不同的阶段（Stage），一个阶段包含一系列函数执行流水线。图中的A、B、C、D、E、F分别代表不同的RDD，RDD内的方框代表分区。数据从HDFS输入Spark，形成RDD A和RDD C，RDD C上执行map操作，转换为RDD D， RDD B和 RDD E执行join操作，转换为F，而在B和E连接转化为F的过程中又会执行Shuffle，最后RDD F 通过函数saveAsSequenceFile输出并保存到HDFS或 Hbase中

## wordcount示例

``` scala
package com.spark.app
import org.apache.spark.{SparkContext, SparkConf}
object WordCount {
  def main(args: Array[String]) {
    /**
      * 第1步；创建Spark的配置对象SparkConf，设置Spark程序运行时的配置信息
      * 例如 setAppName用来设置应用程序的名称，在程序运行的监控界面可以看到该名称，
      * setMaster设置程序运行在本地还是运行在集群中，运行在本地可是使用local参数，也可以使用local[K]/local[*],
      * 可以去spark官网查看它们不同的意义。 如果要运行在集群中，以Standalone模式运行的话，需要使用spark://HOST:PORT
      * 的形式指定master的IP和端口号，默认是7077
      */
    val conf = new SparkConf().setAppName("WordCount").setMaster("local")
//  val conf = new SparkConf().setAppName("WordCount").setMaster("spark://master:7077")  // 运行在集群中

    /**
      * 第2步：创建SparkContext 对象
      * SparkContext是Spark程序所有功能的唯一入口
      * SparkContext核心作用： 初始化Spark应用程序运行所需要的核心组件，包括DAGScheduler、TaskScheduler、SchedulerBackend
      * 同时还会负责Spark程序往Master注册程序
      *
      * 通过传入SparkConf实例来定制Spark运行的具体参数和配置信息
      */
    val sc = new SparkContext(conf)

    /**
      * 第3步： 根据具体的数据来源(HDFS、 HBase、Local FS、DB、 S3等)通过SparkContext来创建RDD
      * RDD 的创建基本有三种方式： 根据外部的数据来源(例如HDFS)、根据Scala集合使用SparkContext的parallelize方法、
      * 由其他的RDD操作产生
      * 数据会被RDD划分成为一系列的Partitions，分配到每个Partition的数据属于一个Task的处理范畴
      */

    val lines = sc.textFile("D:/resources/README.md")   // 读取本地文件
//  val lines = sc.textFile("/library/wordcount/input")   // 读取HDFS文件，并切分成不同的Partition
//  val lines = sc.textFile("hdfs://master:9000/libarary/wordcount/input")  // 或者明确指明是从HDFS上获取数据

    /**
      * 第4步： 对初始的RDD进行Transformation级别的处理，例如 map、filter等高阶函数来进行具体的数据计算
      */
    val words = lines.flatMap(_.split(" ")).filter(word => word != " ")  // 拆分单词，并过滤掉空格，当然还可以继续进行过滤，如去掉标点符号

    val pairs = words.map(word => (word, 1))  // 在单词拆分的基础上对每个单词实例计数为1, 也就是 word => (word, 1)

    val wordscount = pairs.reduceByKey(_ + _)  // 在每个单词实例计数为1的基础之上统计每个单词在文件中出现的总次数, 即key相同的value相加
//  val wordscount = pairs.reduceByKey((v1, v2) => v1 + v2)  // 等同于

    wordscount.collect.foreach(println)  // 打印结果，使用collect会将集群中的数据收集到当前运行drive的机器上，需要保证单台机器能放得下所有数据

    sc.stop()   // 释放资源

  }
}
```

