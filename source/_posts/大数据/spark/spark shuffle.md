---
title: spark shuffle
date: 2018/5/12 15:30:25
category:
- 大数据
- spark
tag:
- spark
comments: true  
---

# spark shuffle

对于大数据计算框架而言，Shuffle阶段的设计优劣是决定性能好坏的关键因素之一。 

**shuffle基本概念** 
shuffle是一个算子，表达的是多对多的依赖关系，在MapReduce计算框架中，是连接Map阶段和Reduce阶段的纽带，即每个Reduce Task从每个Map Task产生数的据中读取一片数据。 
通常shuffle分为两部分：Map阶段的数据准备和Reduce阶段的数据拷贝。

**Map阶段的数据准备** 
Map阶段需根据Reduce阶段的Task数量决定每个Map Task输出的数据分片数目，有多种方式存放这些数据分片，不同的数据存放方式各有优缺点和适用场景。

一般而言，shuffle在Map端的数据要存储到磁盘上，以防止容错触发重算带来的庞大开销（如果保存到Reduce端内存中，一旦Reduce Task挂掉了，所有Map Task需要重算）。

数据在磁盘上存放方式有多种可选方案，在MapReduce前期设计中，每个Map Task为每个Reduce Task产生一个文件，该文件只保存特定Reduce Task需处理的数据，这样会产生M*R个文件，如果M和R非常庞大，比如均为1000，则会产生100w个文件，产生和读取这些文件会产生大量的随机IO，效率非常低下。解决这个问题的一种直观方法是减少文件数目，常用的方法有：

- 将一个节点上所有Map产生的文件合并成一个大文件（MapReduce现在采用的方案）， 

- 每个节点产生{(slot数目)*R}个文件（Spark优化后的方案）。 

   不管是MapReduce 1.0还是Spark，每个节点的资源会被抽象成若干个slot，由于一个Task占用一个slot，因此slot数目可看成是最多同时运行的Task数目。如果一个Job的Task数目非常多，限于slot数目有限，可能需要运行若干轮。这样，只需要由第一轮产生{(slot数目)*R}个文件，后续几轮产生的数据追加到这些文件末尾即可。因此，后一种方案可减少大作业产生的文件数目。

**Reduce阶段的数据拷贝** 
在Reduce端，各个Task会并发启动多个线程同时从多个Map Task端拉取数据。 
在Reduce阶段的主要任务是对数据进行按组规约。也就是说，需要将数据分成若干组，以便以组为单位进行处理。大家知道，分组的方式非常多，常见的有： 

- Map/HashTable（key相同的，放到同一个value list中）； 
- Sort（按key进行排序，key相同的一组，经排序后会挨在一起）。 

这两种方式各有优缺点： 
第一种复杂度低，效率高，但是需要将数据全部放到内存中； 
第二种方案复杂度高，但能够借助磁盘（外部排序）处理庞大的数据集。

Spark前期采用了第一种方案，而在最新的版本中加入了第二种方案，Hadoop MapReduce则从一开始就选用了基于sort的方案。下面将对其进行详细分析。

### 什么是Spark的Shuffle

![这里写图片描述](https://img-blog.csdn.net/20160916114543412) 

Spark有很多算子，比如：groupByKey、join等等都会产生shuffle。 
产生shuffle的时候，首先会产生Stage划分。 

上一个Stage会把 计算结果放在LocalSystemFile中，并汇报给Driver； 
下一个Stage的运行由Driver触发，Executor向Driver请求，把上一个Stage的计算结果抓取过来。

## Hadoop MapReduce Shuffle发展史

![这里写图片描述](https://img-blog.csdn.net/20160916121108219) 

该图表达了**Hadoop**的map和reduce两个阶段，**通过Shuffle怎样把map task的输出结果有效地传送到reduce端，描述着数据从map task输出到reduce task输入的这段过程**。 
map的计算为reduce产生不同的文件，在Hadoop集群环境中，大部分map task与reduce task的执行是在不同的节点上，reduce执行时需要跨节点去拉取其它节点上的map task结果，那么对集群内部的网络资源消耗会很严重。我们希望最大化地减少不必要的消耗, 于是对Shuffle过程的期望有： 

- 完整地从map task端拉取数据到reduce 端。 
- 在跨节点拉取数据时，尽可能地减少对带宽的不必要消耗。 
- 减少磁盘IO对task执行的影响。 
可优化的地方主要在于减少拉取数据的量及尽量使用内存而不是磁盘。

### map端的Shuffle细节：

整个map流程，简单些可以这样说： 
1）input, 根据split输入数据，运行map任务； 
2）patition, 每个map task都有一个内存缓冲区，存储着map的输出结果； 
3）spill, 当缓冲区快满的时候需要将缓冲区的数据以临时文件的方式存放到磁盘； 
4）merge, 当整个map task结束后再对磁盘中这个map task产生的所有临时文件做合并，生成最终的正式输出文件，然后等待reduce task来拉数据。

**map流程的细节：** 
输入数据：在Map Reduce中，map task只读取split，Split与block的对应关系可能是多对一，默认是一对一； 

1. mapper运行后，通过Partitioner接口，根据key或value及reduce的数量来决定当前map的输出数据最终应该交由哪个reduce task处理。然后将数据写入内存缓冲区中，缓冲区的作用是批量收集map结果，减少磁盘IO的影响。我们的key/value对以及Partition的结果都会被写入缓冲区。当然写入之前，key与value值都会被序列化成字节数组； 
2. 内存缓冲区有大小限制，默认是100MB。需要在一定条件下将缓冲区中的数据临时写入磁盘，从内存往磁盘写数据的过程被称为Spill（溢写）； 
3. splill是由单独线程来完成，不影响往缓冲区写map结果的线程,splill的过程会涉及到Sort和Combiner，当splill线程启动后，需要对锁定内存块空间内的key做排序，是对序列化的字节做排序。 如果有很多个key/value对需要发送到某个reduce端去，那么需要将这些key/value值拼接到一块，减少与partition相关的索引记录，非正式地合并数据叫做combine了， Combiner会优化MapReduce的中间结果。 
4. 每次溢写会在磁盘上生成一个溢写文件，如果map的输出结果很大，就会有多个溢写文件存在。当map task完成时，内存缓冲区中的数据也全部溢写到磁盘中形成一个溢写文件。最终磁盘中会至少有一个这样的溢写文件存在(如果map的输出结果很少，当map执行完成时，只会产生一个溢写文件)，因为最终的文件只有一个，所以需要将这些溢写文件归并到一起，这个过程就叫做Merge。

至此，map端的所有工作都已结束，最终生成的这个文件也存放在TaskTracker够得着的某个本地目录内。每个reduce task不断地通过RPC从JobTracker那里获取map task是否完成的信息，如果获知TaskTracker上的map task执行完成，Shuffle的后半段过程开始启动。

### reduce 端的Shuffle细节： 

reduce task在执行之前的工作就是不断地拉取当前job里每个map task的最终结果，然后对从不同地方拉取过来的数据不断地做merge，也最终形成一个文件作为reduce task的输入文件。

1. Copy过程，简单地拉取数据。 

   Reduce进程启动一些数据copy线程(Fetcher)，通过HTTP方式请求map task所在的TaskTracker获取map task的输出文件。因为map task早已结束，这些文件就归TaskTracker管理在本地磁盘中。

2. Merge阶段。 

   这里的merge如map端的merge动作，只是数组中存放的是不同map端copy来的数值。Copy过来的数据会先放入内存缓冲区中，这里的缓冲区大小要比map端的更为灵活，它基于JVM的heap size设置，因为Shuffle阶段Reducer不运行，所以应该把绝大部分的内存都给Shuffle用。merge有三种形式：1)内存到内存 2)内存到磁盘 3)磁盘到磁盘。默认情况下第一种形式不启用。当内存中的数据量到达阈值，就启动内存到磁盘的merge。 
   与map 端类似，这也是溢写的过程，然后在磁盘中生成了众多的溢写文件。第二种merge方式一直在运行，直到没有map端的数据时才结束，然后启动第三种磁盘到磁盘的merge方式生成最终的那个文件。

3. Reducer的输入文件。 

   不断地merge后，最后会生成一个“最终文件”。为什么加引号？因为这个文件可能存在于磁盘上，也可能存在于内存中。对我们来说，当然希望它存放于内存中，直接作为Reducer的输入，但默认情况下，hadoop是把这个文件是存放于磁盘中的。当Reducer的输入文件已定，整个Shuffle才最终结束。然后就是Reducer执行，把结果放到HDFS上。

### Hadoop的MapReduce Shuffle数据流动过程

![这里写图片描述](https://img-blog.csdn.net/20160916121058297) 

图上map阶段，有4个map；Reduce端，有3个reduce。 
4个map 也就是4个JVM，每个JVM处理一个数据分片（split1~split4），每个map产生一个map输出文件，但是每个map都为后面的reduce产生了3部分数据（分别用红1、绿2、蓝3标识），也就是说每个输出的map文件都包含了3部分数据。正如前面第二节所述： 
mapper运行后，通过Partitioner接口，根据key或value及reduce的数量来决定当前map的输出数据最终应该交由哪个reduce task处理 
Reduce端一共有3个reduce，去前面的4个map的输出结果中抓取属于自己的数据。

## Spark Shuffle

![这里写图片描述](https://img-blog.csdn.net/20160914195030665) 

该图描述了最简单的Spark 0.X版本的Spark Shuffle过程。 
与Hadoop Map Reduce的区别在于输出文件个数的变化。

每个ShuffleMapTask产生与Ruducer个数相同的Shuffle blockFile文件，图中有3个reducer，那么每个ShuffleMapTask就产生3个Shuffle blockFile文件，4个ShuffleMapTask，那么一共产生12个Shuffle blockFile文件。 

在内存中每个Shuffle blockFile文件都会存在一个句柄从而消耗一定内存，又因为物理内存的限制，就不能有很多并发，这样就限制了Spark集群的规模。 
该图描绘的只是Spark 0.X版本而已，让人误以为Spark不支持大规模的集群计算，当时这只是**Hash Based Shuffle**。Spark后来做了改进，引入了**Sort Based Shuffle**之后，就再也没有人说Spark只支持小规模的集群运算了。

Shuffle Read在几种模式中都差不多

### Shuffle Read

回忆一下，每个Stage的上边界，要么需要从外部存储读取数据，要么需要读取上一个Stage的输出；而下边界，要么是需要写入本地文件系统（需要Shuffle），以供childStage读取，要么是最后一个Stage，需要输出结果。这里的Stage，在运行时的时候就是可以以pipeline的方式运行的一组Task，除了最后一个Stage对应的是ResultTask，其余的Stage对应的都是ShuffleMap Task。

而除了需要从外部存储读取数据和RDD已经做过cache或者checkpoint的Task，一般Task的开始都是从ShuffledRDD的ShuffleRead开始的。

如果在本地有，那么可以直接从BlockManager中获取数据；如果需要从其他的节点上获取，那么需要走网络。

读取数据的主要流程：

1. 获取待拉取数据的iterator；
2. 使用AppendOnlyMap/ExternalAppendOnlyMap 做combine，这个过程和shuffle write一样；
3. 如果需要对key排序，则使用ExternalSorter。





### Hash based shuffle

Hash based shuffle的==每个mapper都需要为每个reducer写一个文件，供reducer读取，即需要产生M*R个数量的文件==，如果mapper和reducer的数量比较大，产生的文件数会非常多。 
Hadoop Map Reduce被人诟病的地方，很多不需要sort的地方的sort导致了不必要的开销，于是Spark的==Hash based shuffle设计的目标之一就是避免不需要的排序，但是它在处理超大规模数据集的时候，产生了大量的磁盘IO和内存的消耗，很影响性能。== 
Hash based shuffle不断优化，Spark 0.8.1引入的file consolidation在一定程度上解决了这个问题。

> - **不需要排序的Hash Shuffle是否一定比需要排序的Sorted Shuffle速度更快？**
>
>   ​	不一定！如果数据规模比较小的情形下，Hash Shuffle会比Sorted Shuffle速度快（很多）！但是如果数据量大，此时Sorted Shuffle一般都会比Hash Shuffle快（很多），因为如果数据规模比较大，Hash Shuffle甚至无法处理，因为Hash Shuffle会产生很多的句柄，小文件，这时候磁盘和内存会变成瓶颈，而Sorted Shuffle就会极大的节省内存和磁盘的访问，所以更有利于更大规模的计算。Hash Shuffle适合中小型规模的数据计算。 

每个ShuffleMapTask会根据key的哈希值计算出当前的key需要写入的Partition，然后把决定后的结果写入单独的文件，此时会导致==每个Task产生R（指下一个Stage的Task并行度）个文件==，如果当前的Stage中有M个ShuffleMapTask，则会产生M*R个文件！！！此时数据已经分好类了，==下一个Stage就会通过网络根据Driver端的注册信息，因为上一个Stage写过的内容会注册给Driver，然后向Driver获取上一个Stage的输出位置==，就会通过网络去读取数据，数据分成几种类型只跟下一个阶段分成多少个任务有关系，因为下一个阶段的任务数表示数据被分成多少类。跟并行度没有关系，也就是说跟实际并行运行多少任务没有关系。 
注意：Shuffle操作绝大多数情况下都要通过网络，如果Mapper和Reducer在同一台机器上，此时只需要读取本地磁盘即可。Spark中的Executor是线程池中的线程复用的，这个线程有可能运行上一个Stage的Task，也有可能运行下一个Stage的Task。 

#### Hash Shuffle Writer实现解析 

Shuffle Map Task计算是调用ShuffleMapTask.runTask执行的。 

核心代码如下： 

1. 获得ShuffleManager为 Hash Shuffle。 
2. 获得Hash Shuffle的Writer方法：HashShuffleWriter. 
3. 调用HashShuffleWriter的write方法。其中调用RDD的iterator方法计算，然后将结果传入给write。

``` scala
val manager = SparkEnv.get.shuffleManager//获得ShuffleManager
//获得Hash Shuffle的Writer方法
writer = manager.getWriter[Any, Any](dep.shuffleHandle, partitionId, context)
//调用HashShuffleWriter的write方法，
writer.write(rdd.iterator(partition, context).asInstanceOf[Iterator[_ <: Product2[Any, Any]]])
```

下面具体看write方法。

```scala
/** Write a bunch of records to this task's output */
override def write(records: Iterator[Product2[K, V]]): Unit = {
//判断aggregator是否被定义
  val iter = if (dep.aggregator.isDefined) {
//判断数据是否需要聚合如果需要，聚合records
    if (dep.mapSideCombine) {
      dep.aggregator.get.combineValuesByKey(records, context)
//中间代码省略
//elem是(K,V)形式的，通过K计算出bucketId
  for (elem <- iter) {
    val bucketId = dep.partitioner.getPartition(elem._1)
//然后再通过bucketId具体写入那个partition
//此时Shuffle是FileShuffleBlockResolver
    shuffle.writers(bucketId).write(elem._1, elem._2)
  }
```

具体看一下FileShuffleBlockResolver.writers：

``` scala
val writers: Array[DiskBlockObjectWriter] = {
  Array.tabulate[DiskBlockObjectWriter](numReducers) { bucketId =>
    val blockId = ShuffleBlockId(shuffleId, mapId, bucketId)
    val blockFile = blockManager.diskBlockManager.getFile(blockId)
    val tmp = Utils.tempFileWith(blockFile)
//tmp也就是blockFile如果已经存在则，在后面追加数据
    blockManager.getDiskWriter(blockId, tmp, serializerInstance, bufferSize, writeMetrics)
  }
```

blockManager.getDiskWriter就会为每个文件创建一个DiskBlockObjectWriter

DiskBlockObjectWriter可以直接向一个在磁盘上的文件写数据，并且允许在后面追加数据

#### Hash Shuffle的两大死穴：

1. Shuffle前会产生海量的小文件于磁盘之上，此时会产生大量耗时低效的IO操作； 
2. 内存不够用！！！由于内存中需要保存海量的文件操作句柄和临时缓存信息，如果数据处理规模比较庞大的话，内存不可承受，出现OOM等问题！ 

进行HashShuffle的时候会根据后面的Task数，生成对应数量的小文件，而每个小文件也就是一种类型，在数据处理的时候，Task就从前面的小文件抓取需要的数据即可，它会导致同时打开过多的文件，这样就会占用过多的内存，写文件通过Write Handler默认是50KB。 

##### Consalidate

为了改善上述的问题（同时打开过多文件导致Writer Handler内存使用过大以及产生过度文件导致大量的随机读写带来的效率极为低下的磁盘IO操作），Spark后来推出了Consalidate机制，来把小文件(指的是每个Map都要为所有的Reducer产生Reducer个数的小文件)合并，此时Shuffle时文件产生的数量为cores*R，对于ShuffleMapTask的数量明显多于同时可用的并行Cores的数量的情况下，Shuffle产生的文件会大幅度减少，会极大降低OOM的可能； 

**Consalidate机制**：把同一个Task的输出变成一个文件进行合并，根据CPU的个数来决定具体产生多少文件，对于运行在同一个core的Shuffle Map Task，第一个Shuffle Map Task会创建一个，之后的就会将数据追加到这个文件上而不是新建一个文件。 

但是在生成环境下，并行度特别大的话，还是会产生原来的问题。 
为此Spark推出了Shuffle Pluggable开放框架，方便系统升级的时候定制Shuffle功能模块，也方便第三方系统改造人员根据实际的业务场景来开发具体最佳的Shuffle模块；核心接口ShuffleManager，具体默认实现有HashShuffleManager、SortShuffleManager等

### Sort based shuffle

为了解决hash based shuffle性能差的问题，Spark 1.1 引入了**Sort based shuffle**，完全借鉴map reduce实现，每个Shuffle Map Task只产生一个文件，不再为每个Reducer生成一个单独的文件，将所有的结果只写到一个Data文件里，同时生成一个index文件，index文件存储了Data中的数据是如何进行分类的。Reducer可以通过这个index文件取得它需要处理的数据。 下一个Stage中的Task就是根据这个Index文件来获取自己所要抓取的上一个Stage中的Shuffle Map Task的输出数据。

Shuffle Map Task产生的结果只写到一个Data文件里, 避免产生大量的文件，从而节省了内存的使用和顺序Disk IO带来的低延时。节省内存的使用可以减少GC的风险和频率。 而减少文件的数量可以避免同时写多个文件对系统带来的压力。 
Sort based shuffle在速度和内存使用方面也优于Hash based shuffle。 
以上逻辑可以使用下图来描述：

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fre7raa5w2j305v0c1gmc.jpg) 

Sort based Shuffle包含两阶段的任务： 

- **产生Shuffle数据的阶段(Map阶段)** 

  Write 对应的是ShuffleMapTask,具体的写操作ExternalSorter来负责，数据可以通过BlockManager写在内存、磁盘以及Tachyon等，例如想非常快的Shuffle，此时考虑可以把数据写在内存中，但是内存不稳定，建议采用内存+磁盘。 

- **使用Shuffle数据的阶段(Reduce阶段)** 

  需要实现ShuffleManager的getReader，Reader会向Driver去获取上一个Stage产生的Shuffle数据)。Read 阶段由ShuffleRDD里的HashShuffleReader来完成。如果拉来的数据如果过大，需要落地，则也由ExternalSorter来完成的

#### 具体实现

1. ShuffleMapTask会根据Key，相应的Partition进行Sort，==如果属于同一个Partition本身不会进行Sort。==
2. 在进行Sort的时候如果内存不够用的话，它会将哪些已经排序的数据写入到外部磁盘，结束的时候再进行归并排序，这时候基本上就不受内存限制了，同时由于在一个文件中，在一个File中，分为File不同的segment，为了高效的读取不同的segment，它就有一个index文件，会记录不同的Partition信息，BlockManager也会对它的寻址算法进行优化。 
3. Sort Shuffle Writer对于每个Partition会创建一个数组，存储他所包含的Key,Value.每个需要处理的Key,Value,都会被插入到相应的数组中，如果数组的大小超过了具体规定大小的限定值的时候，就需要将内容写入到外部存储。
4. 文件开始的部分记录我们这个Partition的ID，以及这个文件具体保存了有哪些数量的数据。
5. 最后将所有写入到外部文件进行归并排序，归并排序的时候文件不能过多，如果过多的话会消耗很多的内存，可能会有OOM的风险，或者垃圾回收过多的风险，过少性能不好，会有延迟，最优一般同时打开10~100个文件。
6. 最后生成文件的时候需要生成index索引文件，由于它已经排序好了索引文件，所以Reducer去读取索引文件的时候就会非常方便。

#### 排序过程详解

==Spark基于Sorted-Based Shuffle 它产出的结果是有序的。是错误的==

Sorted-Based Shuffle 的核心是借助于 ExternalSorter 把每个 ShuffleMapTask 的输出，排序到一个文件中 (FileSegmentGroup)，为了区分下一个阶段 Reducer Task 不同的内容，它还需要有一个索引文件 (Index) 来告诉下游 Stage 的并行任务，那一部份是属于你的。

![img](https://img-blog.csdn.net/20170505064115674?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZHVhbl96aGlodWE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center) 

Shuffle Map Task 在ExternalSorter 溢出到磁盘的时候，产生一组 File （File Group是hashShuffle中的概念，理解为一个file文件池，这里为区分，使用File的概念，==FileSegment根据PartionID排序）==和 一个索引文件，File 里的 FileSegement 会进行排序，在 Reducer 端有4个Reducer Task，下游的 Task 可以很容易跟据索引 (index) 定位到这个 Fie 中的哪部份 FileSegement 是属于下游的，它相当于一个指针，下游的 Task 要向 Driver 去碓定文件在那里，然后到了这个 File 文件所在的地方，实际上会跟 BlockManager 进行沟通，BlockManager 首先会读一个 Index 文件，根据它的命名则进行解析，比如说下一个阶段的第一个 Task，一般就是抓取第一个 Segment，这是一个指针定位的过程。

**Sort-Based Shuffle 最大的意义是减少临时文件的输出数量，且只会产生两个文件：一个是包含不同内容划分成不同 FileSegment 构成的单一文件 File，另外一个是索引文件 Index。**

 Sort-Based Shuffle  Mapper端的 Sort and Spill 的过程 (ApendOnlyMap时不进行排序，Spill 到磁盘的时候再进行排序的)

ExternalSorter.scala中有2个很重要的数据结构：

``` scala
@volatile private var map = new PartitionedAppendOnlyMap[K, C]
@volatile private var buffer = new PartitionedPairBuffer[K, C]
```

1. 在map端进行combine：PartitionedAppendOnlyMap 是map类型的数据结构，map是key-value ，在本地进行聚合，在本地key值不变，Value不断进行更新
2. 在map端没有combine：使用PartitionedPairBuffer

**排序**：以partition ID进行排序，实现快速的写，方便的读操作；关键的一点对KEY进行操作。

``` scala
private[spark] class PartitionedAppendOnlyMap[K, V]
  extends SizeTrackingAppendOnlyMap[(Int, K), V] with WritablePartitionedPairCollection[K, V] {

  def partitionedDestructiveSortedIterator(keyComparator: Option[Comparator[K]])
    : Iterator[((Int, K), V)] = {
    val comparator = keyComparator.map(partitionKeyComparator).getOrElse(partitionComparator)
    destructiveSortedIterator(comparator)
  }

  def insert(partition: Int, key: K, value: V): Unit = {
    update((partition, key), value)
  }
}
```

### Tungsten-sort Based Shuffle

Tungsten-sort 在特定场景下基于现有的Sort Based Shuffle处理流程，对内存/CPU/Cache使用做了非常大的优化。带来高效的同时，也就限定了自己的使用场景，所以Spark 默认开启的还是Sort Based Shuffle。

> Tungsten 是钨丝的意思。 Tungsten Project 是 Databricks 公司提出的对Spark优化内存和CPU使用的计划，该计划初期对Spark SQL优化的最多，不过部分RDD API 还有Shuffle也因此受益。

Tungsten-sort是对普通sort的一种优化，排序的不是内容本身，而是内容序列化后字节数组的指针(元数据)，把数据的排序转变为了指针数组的排序，实现了直接对序列化后的二进制数据进行排序。由于直接基于二进制数据进行操作，所以在这里面没有序列化和反序列化的过程。内存的消耗降低，相应的也会减少gc的开销。

Tungsten-sort优化点主要在三个方面:

1. 直接在serialized binary data上进行sort而不是java objects，减少了memory的开销和GC的overhead。
2. 提供cache-efficient sorter，使用一个8 bytes的指针，把排序转化成了一个指针数组的排序。 
3. spill的merge过程也无需反序列化即可完成。

这些优化的实现导致引入了一个新的内存管理模型，类似OS的Page，Page是由MemoryBlock组成的, 支持off-heap(用NIO或者Tachyon管理) 以及 on-heap 两种模式。为了能够对Record 在这些MemoryBlock进行定位，又引入了Pointer的概念。

Sort Based Shuffle里存储数据的对象是PartitionedAppendOnlyMap,这只是一个放在JVM heap里普通对象。在Tungsten-sort中，它被替换成了类似操作系统内存页的对象。如果无法申请新的Page,这个时候就要执行spill溢写操作，将数据写到磁盘。具体触发条件和Sort Based Shuffle 类似。

**开启条件**

Spark 默认开启的是Sort Based Shuffle,想要打开Tungsten-sort ,请设置

```
spark.shuffle.manager=tungsten-sort
```

对应的实现类是：

```
org.apache.spark.shuffle.unsafe.UnsafeShuffleManager
```

数据一旦进来，就使用shuffle write进行序列化，在序列化的二进制基础上进行排序，这样就可以减少内存的GC。这种优化需要序列化器可以在不反序列化的情况下重新排序。

**当且仅当下面条件都满足时，才能使用Tungsten-sort Shuffle:** 

1. Shuffle 文件的数量不能大于 16777216
2. Shuffle 的序列化器需要是 KryoSerializer 或者 Spark SQL’s 自定义的一些序列化方式. 因为整个过程是追求不反序列化的，所以不能做aggregation 
3. Shuffle dependency 不能带有aggregation 或者输出不需要排序 
4. 序列化时，单条记录不能大于 128 MB

> **Tungsten内存模型：**
>
> ![这里写图片描述](https://img-blog.csdn.net/20160920161953798) 
>
> 这张图其实画的是 on-heap 的内存逻辑图，#Page 部分为13bit, Offset 为51bit,你会发现 2^51 >>128M的。但是在Shuffle的过程中，对51bit 做了压缩，使用了27bit
>
> 具体如下：
>
> ```
>  [24 bit partition number][13 bit memory page number][27 bit offset in page]
> ```
>
> 这里预留出的24bit给了partition number,为了后面的排序用。上面的好几个限制其实都是因为这个指针引起的：
>
> 1. 一个是partition 的限制，前面的数字 `16777216` 就是来源于partition number 使用24bit 表示的。
> 2. 第二个是page number
> 3. 第三个是偏移量，最大能表示到2^27=128M。那一个task 能管理到的内存是受限于这个指针的，最多是 2^13 * 128M 也就是1TB左右。
>
> 对于第一个限制，那是因为后续Shuffle Write的sort 部分，只对前面24bit的partiton number 进行排序，key的值没有被编码到这个指针，所以没办法进行ordering
>
> 同时，因为整个过程是追求不反序列化的，所以不能做aggregation。

#### Shuffle Write

数据会通过 `UnsafeShuffleExternalSorter.insertRecordIntoSorter` 一条一条写入到 `serOutputStream` 序列化输出流。

类似于Sort Based Shuffle 中的`ExternalSorter`，在Tungsten Sort 对应的为`UnsafeShuffleExternalSorter`,记录序列化后就通过`sorter.insertRecord`方法放到sorter里去了。

这里sorter 负责申请Page,释放Page,判断是否要进行spill都这个类里完成。代码的架子其实和Sort Based 是一样的。 

![图片来源:<a href=](http://upload-images.jianshu.io/upload_images/1063603-66198ebd48a0a66e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 

## 算子示例

### combineByKey

因为combineByKey是Spark中一个比较核心的高级函数，其他一些高阶键值对函数底层都是用它实现的。诸如 groupByKey,reduceByKey等等

``` scala
def combineByKey[C](  
      createCombiner: V => C,  
      mergeValue: (C, V) => C,  
      mergeCombiners: (C, C) => C,  
      partitioner: Partitioner,  
      mapSideCombine: Boolean = true,  
      serializer: Serializer = )  
```

- **createCombiner**: combineByKey() 会遍历分区中的所有元素，因此每个元素的键要么还没有遇到过，要么就 和之前的某个元素的键相同。如果这是一个新的元素， combineByKey() 会使用一个叫作 createCombiner() 的函数来创建 那个键对应的累加器的初始值
- **mergeValue:** 如果这是一个在处理当前分区之前已经遇到的键， 它会使用 mergeValue() 方法将该键的累加器对应的当前值与这个新的值进行合并
- **mergeCombiners:** 由于每个分区都是独立处理的， 因此对于同一个键可以有多个累加器。如果有两个或者更 多的分区都有对应同一个键的累加器， 就需要使用用户提供的mergeCombiners() 方法将各 个分区的结果进行合并。
- **partitioner**：Partitioner（分区器），Shuffle时需要通过Partitioner的分区策略进行分区。

#### 执行过程

1. shuffle write操作中，遇到新<K,V>，通过createCombiner生成累加器，遇到已经出现过的<K,V>，通过mergeValue进行合并，在这个过程中使用AppendOnlyMap/ExternalAppendOnlyMap 做combine，即相同的key合并
2. shuffle read操作中，通过mergeCombiners将不同分区mergeValue后的结果进行合并，获取数据后也会使用AppendOnlyMap/ExternalAppendOnlyMap 做combine，即相同的key合并

### sortByKey

sortByKey函数作用于Key-Value形式的RDD，并对Key进行排序。

```scala
def sortByKey(ascending: Boolean = true, numPartitions: Int = self.partitions.size)
    : RDD[(K, V)] =
{
  val part = new RangePartitioner(numPartitions, self, ascending)
  new ShuffledRDD[K, V, V](self, part)
    .setKeyOrdering(if (ascending) ordering else ordering.reverse)
}

```

该函数返回的RDD一定是ShuffledRDD类型的，因为对源RDD进行排序，必须进行Shuffle操作，而Shuffle操作的结果RDD就是ShuffledRDD。其实这个函数的实现很优雅，里面用到了RangePartitioner，它可以使得相应的范围Key数据分到同一个partition中，然后内部用到了mapPartitions对每个partition中的数据进行排序，而每个partition中数据的排序用到了标准的sort机制，避免了大量数据的shuffle。

