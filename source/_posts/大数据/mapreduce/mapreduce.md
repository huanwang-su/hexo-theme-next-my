---
title: MapReduce介绍
date: 2018/3/15 20:46:25
category:
- 大数据
- MapReduce
tag:
- MapReduce
comments: true  
---

# 1 MapReduce原理

Mapreduce是一个分布式运算程序的编程框架，是用户开发“基于hadoop的数据分析应用”的核心框架；
Mapreduce核心功能是将用户编写的业务逻辑代码和自带默认组件整合成一个完整的分布式运算程序，并发运行在一个hadoop集群上；

## 1.1 为什么要MAPREDUCE

1. 海量数据在单机上处理因为硬件资源限制，无法胜任
2. 而一旦将单机版程序扩展到集群来分布式运行，将极大增加程序的复杂度和开发难度
3. 引入mapreduce框架后，开发人员可以将绝大部分工作集中在业务逻辑的开发上，而将分布式计算中的复杂性交由框架来处理

>单机版：内存受限，磁盘受限，运算能力受限
>
>分布式：
>
>1. 文件分布式存储（HDFS）
>2. 运算逻辑需要至少分成2个阶段（一个阶段独立并发，一个阶段汇聚）
>3. 运算程序如何分发
>4. 程序如何分配运算任务（切片）
>5. 两阶段的程序如何启动？如何协调？
>
>整个程序运行过程中的监控？容错？重试？
>
>mapreduce就是这样一个分布式程序的通用框架，其应对以上问题的整体结构如下：
>
>`1. MRAppMaster(mapreduceapplication master)
>
>2. MapTask
>3. ReduceTask

## 1.2 MAPREDUCE框架结构及核心运行机制

### 1.2.1 结构

一个完整的mapreduce程序在分布式运行时有三类实例进程：

1. MRAppMaster：负责整个程序的过程调度及状态协调
2. mapTask：负责map阶段的整个数据处理流程
3. ReduceTask：负责reduce阶段的整个数据处理流程

### 1.2.2 MR程序运行流程

#### 1.2.2.1 流程示意图

![](http://ww1.sinaimg.cn/large/0063bT3gly1fmcpo8i081j30xy0kmq42.jpg)

1.  一个mr程序启动的时候，最先启动的是MRAppMaster，MRAppMaster启动后根据本次job的描述信息，计算出需要的maptask实例数量，然后向集群申请机器启动相应数量的maptask进程
2. maptask进程启动之后，根据给定的数据切片范围进行数据处理，主体流程为：
   1. 利用客户指定的inputformat来获取RecordReader读取数据，形成输入KV对
   2. 将输入KV对传递给客户定义的map()方法，做逻辑运算，并将map()方法输出的KV对收集到缓存
   3. 将缓存中的KV对按照K分区排序后不断溢写到磁盘文件
3. MRAppMaster监控到所有maptask进程任务完成之后，会根据客户指定的参数启动相应数量的reducetask进程，并告知reducetask进程要处理的数据范围（数据分区）
4. Reducetask进程启动之后，根据MRAppMaster告知的待处理数据所在位置，从若干台maptask运行所在机器上获取到若干个maptask输出结果文件，并在本地进行重新归并排序，然后按照相同key的KV为一个组，调用客户定义的reduce()方法进行逻辑运算，并收集运算输出的结果KV，然后调用客户指定的outputformat将结果数据输出到外部存储

## 1.3 MapTask并行度决定机制

maptask的并行度决定map阶段的任务处理并发度，进而影响到整个job的处理速度那么，mapTask并行实例是否越多越好呢？其并行度又是如何决定呢？

### 1.3.1 mapTask并行度的决定机制

一个job的map阶段并行度由客户端在提交job时决定

而客户端对map阶段并行度的规划的基本逻辑为：

将待处理数据执行逻辑切片（即按照一个特定切片大小，将待处理数据划分成逻辑上的多个split），然后**每一个split分配一个mapTask并行实例处理**

这段逻辑及形成的切片规划描述文件，由FileInputFormat实现类的getSplits()方法完成，其过程如下图：

### ![](http://ww1.sinaimg.cn/large/0063bT3gly1fmcq54zsybj30rq0h00tg.jpg)1.3.2 FileInputFormat切片机制

1. 切片定义在InputFormat类中的getSplit()方法

2. FileInputFormat中默认的切片机制：

   a)	简单地按照文件的内容长度进行切片
   b)	切片大小，默认等于block大小
   c)	切片时不考虑数据集整体，而是逐个针对每一个文件单独切片

3. FileInputFormat中切片的大小的参数配置

   在FileInputFormat中，计算切片大小的逻辑：Math.max(minSize, Math.min(maxSize, blockSize));  切片主要由这几个值来运算决定

   | minsize：默认值：1            配置参数： mapreduce.input.fileinputformat.split.minsize |
   | ---------------------------------------- |
   | maxsize：默认值：Long.MAXValue         配置参数：mapreduce.input.fileinputformat.split.maxsize |
   | blocksize                                |

4. 选择并发数的影响因素：

   1. 运算节点的硬件配置
   2. 运算任务的类型：CPU密集型还是IO密集型
   3. 运算任务的数据量

## 1.3.3 map并行度的经验之谈

如果硬件配置为2*12core + 64G，恰当的map并行度是大约每个节点20-100个map，最好每个map的执行时间至少一分钟

- 如果job的每个map或者 reduce task的运行时间都只有30-40秒钟，那么就减少该job的map或者reduce数，每一个task(map|reduce)的setup和加入到调度器中进行调度，这个中间的过程可能都要花费几秒钟
- 如果input的文件非常的大，比如1TB，可以考虑将hdfs上的每个block
  size设大，比如设成256MB或者512MB

## 1.4 ReduceTask并行度的决定

reducetask的并行度同样影响整个job的执行并发度和执行效率，但与maptask的并发数由切片数决定不同，Reducetask数量的决定是可以直接手动设置：

``` java
//默认值是1，手动设置为4
job.setNumReduceTasks(4);
```

如果数据分布不均匀，就有可能在reduce阶段产生数据倾斜

> 注意： reducetask数量并不是任意设置，还要考虑业务逻辑需求，有些情况下，需要计算全局汇总结果，就只能有1个reducetask

# 2 MapReduce实践

## 2.1 MAPREDUCE示例编写及编程规范

### 2.1.1 编程规范

（1）用户编写的程序分成三个部分：Mapper，Reducer，Driver(提交运行mr程序的客户端)

（2）Mapper的输入数据是KV对的形式（KV的类型可自定义）

（3）Mapper的输出数据是KV对的形式（KV的类型可自定义）

（4）Mapper中的业务逻辑写在map()方法中

（5）map()方法（maptask进程）对每一个<K,V>调用一次

（6）Reducer的输入数据类型对应Mapper的输出数据类型，也是KV

（7）Reducer的业务逻辑写在reduce()方法中

（8）Reducetask进程对每一组相同k的<k,v>组调用一次reduce()方法

（9）用户自定义的Mapper和Reducer都要继承各自的父类

（10）整个程序需要一个Drvier来进行提交，提交的是一个描述了各种必要信息的job对象

### 2.1.2 wordcount示例编写

定义一个mapper类

``` java
//首先要定义四个泛型的类型
//keyin:  LongWritable    valuein: Text
//keyout: Text            valueout:IntWritable

public class WordCountMapper extends Mapper<LongWritable, Text, Text, IntWritable>{
	//map方法的生命周期：  框架每传一行数据就被调用一次
	//key :  这一行的起始点在文件中的偏移量
	//value: 这一行的内容
	@Override
	protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
		//拿到一行数据转换为string
		String line = value.toString();
		//将这一行切分出各个单词
		String[] words = line.split(" ");
		//遍历数组，输出<单词，1>
		for(String word:words){
			context.write(new Text(word), new IntWritable(1));
		}
	}
}
```

定义一个reducer类

``` java
public class WordCountReduce extends Reduce<Text, IntWritable, Text, IntWritable>{
//生命周期：框架每传递进来一个kv 组，reduce方法被调用一次
	@Override
	protected void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
		//定义一个计数器
		int count = 0;
		//遍历这一组kv的所有v，累加到count中
		for(IntWritable value:values){
			count += value.get();
		}
		context.write(key, new IntWritable(count));
	}
}
```

定义一个主类，用来描述job并提交job

``` java
public class WordCountRunner {
	//把业务逻辑相关的信息（哪个是mapper，哪个是reducer，要处理的数据在哪里，输出的结果放哪里……）描述成一个job对象
	//把这个描述好的job提交给集群去运行
	public static void main(String[] args) throws Exception {
		Configuration conf = new Configuration();
		Job wcjob = Job.getInstance(conf);
		//指定我这个job所在的jar包
//		wcjob.setJar("/home/hadoop/wordcount.jar");
		wcjob.setJarByClass(WordCountRunner.class);
		
		wcjob.setMapperClass(WordCountMapper.class);
		wcjob.setReducerClass(WordCountReducer.class);
		//设置我们的业务逻辑Mapper类的输出key和value的数据类型
		wcjob.setMapOutputKeyClass(Text.class);
		wcjob.setMapOutputValueClass(IntWritable.class);
		//设置我们的业务逻辑Reducer类的输出key和value的数据类型
		wcjob.setOutputKeyClass(Text.class);
		wcjob.setOutputValueClass(IntWritable.class);
		
		//指定要处理的数据所在的位置
		FileInputFormat.setInputPaths(wcjob, "hdfs://hdp-server01:9000/wordcount/data/big.txt");
		//指定处理完成之后的结果所保存的位置
		FileOutputFormat.setOutputPath(wcjob, new Path("hdfs://hdp-server01:9000/wordcount/output/"));
		
		//向yarn集群提交这个job
		boolean res = wcjob.waitForCompletion(true);
		System.exit(res?0:1);
	}
}
```



## 2.2 MapReduce程序运行模式

### 2.2.1 本地运行 

1. mapreduce程序是被提交给LocalJobRunner在本地以单进程的形式运行
2. 而处理的数据及输出结果可以在本地文件系统，也可以在hdfs上
3. 怎样实现本地运行？写一个程序，不要带集群的配置文件（本质是你的mr程序的conf中是否有mapreduce.framework.name=local以及yarn.resourcemanager.hostname参数）
4. 本地模式非常便于进行业务逻辑的debug，只要在eclipse中打断点即可

### 2.2.2 集群运行模式

1. 将mapreduce程序提交给yarn集群resourcemanager，分发到很多的节点上并发执行

2. 处理的数据和输出结果应该位于hdfs文件系统

3. 提交集群的实现步骤：

   A. 将程序打成JAR包，然后在集群的任意一个节点上用hadoop命令启动

   ​     $ hadoop jar wordcount.jar cn.itcast.bigdata.mrsimple.WordCountDriverinputpath outputpath

   B. 直接在linux的eclipse中运行main方法（项目中要带参数：mapreduce.framework.name=yarn以及yarn的两个基本配置）

   C. 如果要在windows的eclipse中提交job给集群，则要修改YarnRunner类

mapreduce程序在集群中运行时的大体流程：

![](http://ww1.sinaimg.cn/large/0063bT3gly1fmczkzxrd3j30yq0erq3n.jpg)

2.3 MapReduce中的Combiner

- combiner是MR程序中Mapper和Reducer之外的一种组件

- combiner组件的父类就是Reducer

- combiner和reducer的区别在于运行的位置：

  - Combiner是在每一个maptask所在的节点运行


  - Reducer是接收全局所有Mapper的输出结果；

- combiner的意义就是对每一个maptask的输出进行局部汇总，以减小网络传输量

  具体实现步骤：

  1. 自定义一个combiner继承Reducer，重写reduce方法
  2. 在job中设置：  job.setCombinerClass(CustomCombiner.class)

(5) combiner能够应用的前提是不能影响最终的业务逻辑而且，combiner的输出kv应该跟reducer的输入kv类型要对应起来

# 3 shuffle机制

## 3.1 概述

- mapreduce中，map阶段处理的数据如何传递给reduce阶段，是mapreduce框架中最关键的一个流程，这个流程就叫shuffle；
- shuffle: 洗牌. 发牌——（核心机制：数据分区，排序，缓存）；
- 具体来说：就是将maptask输出的处理结果数据，分发给reducetask，并在分发的过程中，对数据按key进行了分区和排序；

## 3.2 主要流程

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fmd1vvgzfbj30t90f3wmp.jpg)

shuffle是MR处理流程中的一个过程，它的每一个处理步骤是分散在各个map task和reduce task节点上完成的，整体来看，分为3个操作：

1. 分区partition
2. Sort根据key排序
3. Combiner进行局部value的合并

## 3.3 详细流程

1.  maptask收集我们的map()方法输出的kv对，放到内存缓冲区中
2.  从内存缓冲区不断溢出本地磁盘文件，可能会溢出多个文件
3. 多个溢出文件会被合并成大的溢出文件
4.  在溢出过程中，及合并的过程中，都要调用partitoner进行分组和针对key进行排序
5. reducetask根据自己的分区号，去各个maptask机器上取相应的结果分区数据
6.  reducetask会取到同一个分区的来自不同maptask的结果文件，reducetask会将这些文件再进行合并（归并排序）
7. 合并成大文件后，shuffle的过程也就结束了，后面进入reducetask的逻辑运算过程（从文件中取出一个一个的键值对group，调用用户自定义的reduce()方法）

Shuffle中的缓冲区大小会影响到mapreduce程序的执行效率，原则上说，缓冲区越大，磁盘io的次数越少，执行速度就越快 缓冲区的大小可以通过参数调整,  参数：io.sort.mb  默认100M

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fmd20rdgfej31d01fgdk0.jpg)

# 4 MAPREDUCE中的序列化

## 4.1 概述

Java的序列化是一个重量级序列化框架（Serializable），一个对象被序列化后，会附带很多额外的信息（各种校验信息header，继承体系。。。。），不便于在网络中高效传输；所以hadoop自己开发了一套序列化机制（Writable），精简，高效

## 4.2 Jdk序列化和MR序列化之间的比较

一个是readObject和writeObject

一个是自定义流的解析

## 4.3 自定义对象实现MR中的序列化接口

如果需要将自定义的bean放在key中传输，则还需要实现comparable接口，因为mapreduce框中的shuffle过程一定会对key进行排序,此时，自定义的bean实现的接口应该是：

`public class  FlowBean  implements  WritableComparable<FlowBean> `

需要自己实现的方法是：

``` java
	/**
	 * 反序列化的方法，反序列化时，从流中读取到的各个字段的顺序应该与序列化时写出去的顺序保持一致
	 */
	@Override
	public void readFields(DataInput in) throws IOException {	
		upflow = in.readLong();
		dflow = in.readLong();
		sumflow = in.readLong();	
	}

	/**
	 * 序列化的方法
	 */
	@Override
	public void write(DataOutput out) throws IOException {
		out.writeLong(upflow);
		out.writeLong(dflow);
		//可以考虑不序列化总流量，因为总流量是可以通过上行流量和下行流量计算出来的
		out.writeLong(sumflow);
	}
	
	@Override
	public int compareTo(FlowBean o) {
		//实现按照sumflow的大小倒序排序
		return sumflow>o.getSumflow()?-1:1;
	}
```

# 5 MapReduce与YARN

## 5.1 YARN概述

Yarn是一个资源调度平台，负责为运算程序提供服务器运算源，相当于一个分布式的操作系统平台，而mapreduce等运算程序则相当于运行于操作系统之上的应用程序

1.  yarn并不清楚用户提交的程序的运行机制
2.  yarn只提供运算资源的调度（用户程序向yarn申请资源，yarn就负责分配资源）
3.  yarn中的主管角色叫ResourceManager
4.  yarn中具体提供运算资源的角色叫NodeManager
5.  这样一来，yarn其实就与运行的用户程序完全解耦，就意味着yarn上可以运行各种类型的分布式运算程序（mapreduce只是其中的一种），比如mapreduce. storm程序，spark程序，tez ……
6.  所以，spark. storm等运算框架都可以整合在yarn上运行，只要他们各自的框架中有符合yarn规范的资源请求机制即可
7.  Yarn就成为一个通用的资源调度平台，从此，企业中以前存在的各种运算集群都可以整合在一个物理集群上，提高资源利用率，方便数据共享

## 5.2 Yarn中运行运算程序的示例

mapreduce程序的调度过程，如下图

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fmd2bzrq39j31200i8gnf.jpg)

# 6 Mapreduce中的分区Partitioner

Mapreduce中会将map输出的kv对，按照相同key分组，然后分发给不同的reducetask

默认的分发规则为：根据key的hashcode%reducetask数来分发

所以：如果要按照我们自己的需求进行分组，则需要改写数据分发（分组）组件Partitioner

自定义一个CustomPartitioner继承抽象类：Partitioner然后在job对象中，设置自定义partitioner：

` job.setPartitionerClass(CustomPartitioner.class)`

