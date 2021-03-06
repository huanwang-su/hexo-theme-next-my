---
title: MapReduce组件
date: 2018/3/15 20:46:25
category:
- 大数据
- MapReduce
tag:
- MapReduce
comments: true  
---

## 组件

### inputformat

在MR程序的开发过程中，经常会遇到输入数据不是HDFS或者数据输出目的地不是HDFS的，MapReduce的设计已经考虑到这种情况，它为我们提供了两个组建，只需要我们自定义适合的InputFormat和OutputFormat，就可以完成这个需求

MapReduce中Map阶段的数据输入是由InputFormat决定的，我们可以看到除了实现InputFormat抽象类以外，我们还需要自定义InputSplit和自定义RecordReader类，这两个类的主要作用分别是：split确定数据分片的大小以及数据的位置信息，recordReader具体的读取数据。

``` java
public abstract class InputFormat<K, V> {
  public abstract List<InputSplit> getSplits(JobContext context) throws IOException, InterruptedException; // 获取Map阶段的数据分片集合信息
  public abstract RecordReader<K,V> createRecordReader(InputSplit split, TaskAttemptContext context） throws IOException, InterruptedException; // 创建具体的数据读取对象
}
```

1. 自定义InputSplit

```java
public abstract class InputSplit {  
  public abstract long getLength() throws IOException, InterruptedException; // 获取当前分片的长度大小  
  public abstract String[] getLocations() throws IOException, InterruptedException; // 获取当前分片的位置信息  
}
```

2. 自定义RecordReader

```java
public abstract class RecordReader<KEYIN, VALUEIN> implements Closeable {
  public abstract void initialize(InputSplit split, TaskAttemptContext context) throws IOException, InterruptedException; // 初始化，如果在构造函数中初始化了，那么该方法可以为空
  public abstract boolean nextKeyValue() throws IOException, InterruptedException; //是否存在下一个key/value，如果存在返回true。否则返回false。
  public abstract KEYIN getCurrentKey() throws IOException, InterruptedException;  // 获取当然key
  public abstract VALUEIN getCurrentValue() throws IOException, InterruptedException;  // 获取当然value
  public abstract float getProgress() throws IOException, InterruptedException;  // 获取进度信息
  public abstract void close() throws IOException; // 关闭资源
}
```

### RecordReader

### mapper

每个节点计算mapper后汇总到reduce

![img](https://images2015.cnblogs.com/blog/1122015/201704/1122015-20170414111120361-919366924.png)

### Combiner

Combiner 是 MapReduce 程序中 Mapper 和 Reducer 之外的一种组件，它的作用是在 maptask 之后给 maptask 的结果进行局部汇总，以减轻 reducetask 的计算负载，减少网络传输

1. 如何使用combiner

   Combiner 和 Reducer 一样，编写一个类，然后继承 Reducer， reduce 方法中写具体的 Combiner 逻辑，然后在 job 中设置 Combiner 类： job.setCombinerClass(FlowSumCombine.class)

   ![img](https://images2015.cnblogs.com/blog/1122015/201704/1122015-20170417112259149-15579715.png)

2. 使用combiner注意事项  

   - Combiner 和 Reducer 的区别在于运行的位置：
     - Combiner 是在每一个 maptask 所在的节点运行
     - Reducer 是接收全局所有 Mapper 的输出结果
   - Combiner 的输出 kv 应该跟 reducer 的输入 kv 类型要对应起来
   - Combiner 的使用要非常谨慎，因为 Combiner 在 MapReduce 过程中可能调用也可能不调用，可能调一次也可能调多次，所以： Combiner 使用的原则是：有或没有都不能影响业务逻辑，都不能影响最终结果(求平均值时，combiner和reduce逻辑不一样)

### Sort

自定义的 bean 来封装信息，并将 bean 作为 map 输出的 key 来传输 MR 程序在处理数据的过程中会对数据排序(map 输出的 kv 对传输到 reduce 之前，会排序)， 排序的依据是 map 输出的 key， 所以，我们如果要实现自己需要的排序规则，则可以考虑将排序因素放到 key 中，让 key 实现接口： WritableComparable， 然后重写 key 的 compareTo 方法

### Partitioner

MapReduce 中会将 map 输出的 kv 对，按照相同 key 分组，然后分发给不同的 reducetask默认的分发规则为：根据 key 的 hashcode%reducetask 数来分发

![img](https://images2015.cnblogs.com/blog/1122015/201704/1122015-20170417151634243-560530713.png)

- 如果reduceTask的数量>= getPartition的结果数  ，则会多产生几个空的输出文件part-r-000xx
- 如果 1<reduceTask的数量<getPartition的结果数 ，则有一部分分区数据无处安放，会Exception！！！
- 如果 reduceTask的数量=1，则不管mapTask端输出多少个分区文件，最终结果都交给这一个reduceTask，最终也就只会产生一个结果文件 part-r-00000

### GroupingComparator

GroupingComparator是在reduce的 **secondary sort**阶段分组来使用的，GroupComparator是如何对进入reduce函数中的`key   Iterable<value>` 进行影响。可以在自定义的GroupComparator 中确定哪儿些value组成一组(key不同)，进入一个reduce函数

我们需要理清楚的还有map阶段你的几个自定义：

- parttioner中的getPartition()这个是map阶段自定义分区，
- bean中定义CopmareTo()是在溢出和merge时用来来排序的。

### reduce

![img](https://images2015.cnblogs.com/blog/1122015/201704/1122015-20170414111330189-1950573665.png)

### OutputFormat

MapReduce中Reducer阶段的数据输出是由OutputFormat决定的，决定数据的输出目的地和job的提交对象

```java
public abstract class OutputFormat<K, V> { 
  public abstract RecordWriter<K, V> getRecordWriter(TaskAttemptContext context) throws IOException, InterruptedException; // 获取具体的数据写出对象
  public abstract void checkOutputSpecs(JobContext context) throws IOException, InterruptedException; // 检查输出配置信息是否正确
  public abstract OutputCommitter getOutputCommitter(TaskAttemptContext context) throws IOException, InterruptedException; // 获取输出job的提交者对象
}
```

### 序列化

Java 的序列化是一个重量级序列化框架（ Serializable），一个对象被序列化后，会附带很多额 外的信息（各种校验信息， header，继承体系等），不便于在网络中高效传输；所以， hadoop 自己开发了一套序列化机制（ Writable），精简，高效
Hadoop 中的序列化框架已经对基本类型和 null 提供了序列化的实现了。分别是：

![img](https://images2015.cnblogs.com/blog/1122015/201704/1122015-20170417143247102-379411243.png)

- 自定义对象实现mapreduce框架的序列化

  如果需要将自定义的 bean 放在 key 中传输，则还需要实现 Comparable 接口，因为 mapreduce框中的 shuffle 过程一定会对 key 进行排序，此时，自定义的 bean 实现的接口应该是：`public class FlowBean implements WritableComparable<FlowBean>`

  ```java
  public class StudentWritable implements WritableComparable<StudentWritable> {
      private String name;
      private int age;
      public void write(DataOutput out) throws IOException {
          out.writeUTF(this.name);
          out.writeInt(this.age);
      }
      //注意顺序是一样的
      public void readFields(DataInput in) throws IOException {
          this.name = in.readUTF();
          this.age = in.readInt();
      }
      public int compareTo(StudentWritable o) {
          return 0;
      }
  }
  ```

- hadoop序列化的特点

  - 紧凑：高效实用存储空间
  - 快速：读写数据的额外开销小
  - 可扩展：可透明地读取老格式的数据
  - 互操作：支持多语言的交互

### Job

![img](https://images2015.cnblogs.com/blog/1122015/201704/1122015-20170414111348970-1700075486.png)

![img](https://images2015.cnblogs.com/blog/1122015/201704/1122015-20170414111418267-769150125.png)

## 运行原理

### shuffle机制

及map的输出如何汇聚到reduce

![img](http://img.blog.csdn.net/20170603114321528?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdG90b3R1enVvcXVhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

shuffle是MR处理流程中的一个过程，它的每一个处理步骤是分散在各个map task和reduce task节点上完成的，整体来看，分为3个操作：

1. 分区partition
2. Sort根据key排序
3. Combiner进行局部value的合并

#### 详细流程

1. maptask收集我们的map()方法输出的kv对，放到内存缓冲区中
2. 从内存缓冲区不断溢出本地磁盘文件，可能会溢出多个文件
3. 多个溢出文件会被合并成大的溢出文件
4. 在溢出过程中，及合并的过程中，都要调用partitoner进行分组和针对key进行排序
5. reducetask根据自己的分区号，去各个maptask机器上取相应的结果分区数据
6. reducetask会取到同一个分区的来自不同maptask的结果文件，reducetask会将这些文件再进行合并（归并排序）
7. 合并成大文件后，shuffle的过程也就结束了，后面进入reducetask的逻辑运算过程（从文件中取出一个一个的键值对group，调用用户自定义的reduce()方法）

 Shuffle中的缓冲区大小会影响到mapreduce程序的执行效率，原则上说，缓冲区越大，磁盘io的次数越少，执行速度就越快

### mapreduce程序的运行流程（经典面试题）

1. 一个 mr 程序启动的时候，最先启动的是 MRAppMaster， MRAppMaster 启动后根据本次 job 的描述信息，计算出需要的 maptask 实例数量，然后向集群申请机器启动相应数量的 maptask 进程
2. maptask 进程启动之后，根据给定的数据切片(哪个文件的哪个偏移量范围)范围进行数 据处理，主体流程为：
   - 利用客户指定的 inputformat 来获取 RecordReader 读取数据，形成输入 KV 对	
   - 将输入 KV 对传递给客户定义的 map()方法，做逻辑运算，并将 map()方法输出的 KV 对收 集到缓存
   - 将缓存中的 KV 对按照 K 分区排序后不断溢写到磁盘文件 （超过缓存内存写到磁盘临时文件，最后都写到该文件，ruduce 获取该文件后，删除 ）
3. MRAppMaster 监控到所有 maptask 进程任务完成之后（真实情况是，某些 maptask 进 程处理完成后，就会开始启动 reducetask 去已完成的 maptask 处 fetch 数据），会根据客户指 定的参数启动相应数量的 reducetask 进程，并告知 reducetask 进程要处理的数据范围（数据分区）
4. Reducetask 进程启动之后，根据 MRAppMaster 告知的待处理数据所在位置，从若干台 maptask 运行所在机器上获取到若干个 maptask 输出结果文件，并在本地进行重新归并排序， 然后按照相同 key 的 KV 为一个组，调用客户定义的 reduce()方法进行逻辑运算，并收集运算输出的结果 KV，然后调用客户指定的 outputformat 将结果数据输出到外部存储

### maptask并行度决定机制

一个job的map阶段并行度由客户端在提交job时决定

而客户端对map阶段并行度的规划的基本逻辑为：将待处理数据执行逻辑切片（即按照一个特定切片大小，将待处理数据划分成逻辑上的多个split），然后每一个split分配一个mapTask并行实例处理

这段逻辑及形成的切片规划描述文件，由FileInputFormat实现类的getSplits()方法完成

splitSize=max{minSize,min{maxSize,blockSize}} 图中有错

![](http://ww1.sinaimg.cn/large/0063bT3gly1fovfepo5znj30rq0h00tg.jpg)