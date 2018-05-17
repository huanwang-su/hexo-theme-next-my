---
title: hadoop的io
date: 2018/5/11 20:46:25
category:
- 大数据
- MapReduce
tag:
- hadoop
comments: true  
---

# hadoop的IO操作

## 数据完整性

检测数据是否损坏的常用措施是：在数据第一次引入系统时计算校验和并在数据通过一个不可靠的同道进行传输时再一次计算校验和，这样就能发现数据是否损坏。

### HDFS的数据完整性

　![img](https://images2017.cnblogs.com/blog/999804/201710/999804-20171021231106802-1495466168.png)

数据节点负责在存储数据及其校验和之前验证它们收到的数据。 从客户端和其它数据节点复制过来的数据。客户端写入数据并且将它发送到一个数据节点管线中，在==管线的最后一个数据节点验证校验和==。

客户端读取数据节点上的数据时，会验证校验和，将其与数据节点上存储的校验和进行对比。每个数据节点维护一个连续的校验和验证日志，因此它知道每个数据块最后验证的时间。

每个数据节点还会在后台线程运行一个DataBlockScanner（数据块检测程序），定期验证存储在数据节点上的所有块，为了防止物理存储介质中位衰减锁造成的数据损坏。

## 压缩

文件压缩两大好处：减少存储文件所需要的空间且加快了数据在网络上或从磁盘上或到磁盘上的传输速度。

![img](http://images.51cto.com/files/uploadimg/20120423/105731439.jpg)

所有的要锁算法都要权衡时间/空间：压缩和解压缩的速度更快，其代价通常只能节省少量的时间，我们有9个不同的选项来控制压缩时必须考虑的权衡，-1为优化压缩速度，-9优化压缩空间。

当使用MapReduce处理压缩文件时，需要考虑压缩文件的可分割性。mapreduce一般用可分割的算法比较好

### 编码和解码

编码和解码器用以执行压缩解压算法。在Hadoop中，编码和解码是通过一个压缩解码器接口实现的。

CompressionCodec有两个方法轻松地压缩和解压数据。使用createOutputStream(OutputStream out)创建一个CompressionOutputStream，将其以压缩格式写入底层的流。使用createInputStream(InputStream in) 获取一个CompressionInputStream，从底层的流读取未压缩的数据。

``` java
package com.laos.hadoop; 
import org.apache.hadoop.conf.Configuration; 
import org.apache.hadoop.io.IOUtils; 
import org.apache.hadoop.io.compress.CompressionCodec; 
import org.apache.hadoop.io.compress.CompressionOutputStream; 
import org.apache.hadoop.util.ReflectionUtils; 
public class StreamCompressor { 
    public static void main(String[] args) throws Exception { 
        String codecClassname = "org.apache.hadoop.io.compress.GzipCodec"; 
        Class<?> codecClass = Class.forName(codecClassname); 
        Configuration conf = new Configuration(); 
        CompressionCodec codec = (CompressionCodec) ReflectionUtils 
                .newInstance(codecClass, conf); 
         //将读入数据压缩至System.out
        CompressionOutputStream out = codec.createOutputStream(System.out); 
        IOUtils.copyBytes(System.in, out, 4096, false); 
        out.finish(); 
    } 
}

```

在阅读一个压缩文件时，我们可以从扩展名来推断出它的编码/解码器。以.gz结尾的文件可以用GzipCodec来阅读。CompressionCodecFactory提供了getCodec()方法，从而将文件扩展名映射到相应的CompressionCodec。

### 压缩和输入分隔

考虑如何压缩哪些将由MapReduce处理的数据时，考虑压缩格式是否支持分隔很重要。

如果要压缩MapReduce作业的输出，设置mapred.output.compress为true，mapred.output.compression.codec属性指定编码解码器。

如果输入的文件时压缩过的，MapReduce读取时，它们会自动解压，根据文件扩展名来决定使用那一个压缩解码器。

``` java
package com.laos.hadoop; 
import java.io.IOException; 
import org.apache.hadoop.fs.Path; 
import org.apache.hadoop.io.IntWritable; 
import org.apache.hadoop.io.Text; 
import org.apache.hadoop.io.compress.CompressionCodec; 
import org.apache.hadoop.io.compress.GzipCodec; 
import org.apache.hadoop.mapred.FileInputFormat; 
import org.apache.hadoop.mapred.FileOutputFormat; 
import org.apache.hadoop.mapred.JobClient; 
import org.apache.hadoop.mapred.JobConf; 
public class MaxTemperatureWithCompression { 
     public static void main(String[] args) throws IOException { 
         if (args.length != 2) { 
         System.err.println("Usage: MaxTemperatureWithCompression <input path> " + 
         "<output path>"); 
         System.exit(-1); 
         } 
         
         JobConf conf = new JobConf(MaxTemperatureWithCompression.class); conf.setJobName("Max temperature with output compression"); 
         FileInputFormat.addInputPath(conf, new Path(args[0])); 
         FileOutputFormat.setOutputPath(conf, new Path(args[1])); 
         
         conf.setOutputKeyClass(Text.class); 
         conf.setOutputValueClass(IntWritable.class); 
         
         conf.setBoolean("mapred.output.compress", true); 
         conf.setClass("mapred.output.compression.codec", GzipCodec.class, 
         CompressionCodec.class);         
         JobClient.runJob(conf); 
    } 
}
```

## 序列化

序列化就是把内存中的对象，转换成 字节序列（或其他数据传输协议）以便于存储（持久化）和网络传输。 
反序列化就是将收到 字节序列（或其他数据传输协议）或者是硬盘的持久化数据，转换成 内存中的对象。

#### RPC序列化格式要求

在Hadoop中，系统中多个节点上进程间的通信是通过“远程过程调用（RPC）”实现的。RPC协议将消息序列化成 二进制流后发送到远程节点，远程节点将二进制流反序列化为原始信息。通常情况下，RPC序列化格式如下：

1. 紧凑（compact）

   紧凑格式能充分利用网络带宽。

2. 快速（Fast）

   进程间通信形成了分布式系统的骨架，所以需要尽量减少序列化和反序列化的性能开销，这是基本..最基本的。

3. 可扩展（Extensible）

   为了满足新的需求，协议不断变化。所以控制客户端和服务器的过程中，需要直接引进相应的协议。

4. 支持互操作（Interoperable）

   对于某些系统来说，希望能支持以不同语言写的客户端与服务器交互，所以需要设计需要一种特定的格式来满足这一需求。

### **JDK序列化和反序列化** 

Serialization（序列化）是一种将对象转换为字节流；反序列化deserialization是一种将这些字节流生成一个对象。

a）当你想把的内存中的对象保存到一个文件中或者数据库中时候； 
b）当你想用套接字在网络上传送对象的时候； 
c）当你想通过RMI传输对象的时候；

将需要序列化的类实现Serializable接口就可以了，Serializable接口中没有任何方法，可以理解为一个标记，即表明这个类可以序列化。

### Hadoop序列化和反序列化

在hadoop中，hadoop实现了一套自己的序列化框架，相对于JDK比较简洁，在集群信息的传递上速度更快，容量更小。

在Java中将一个类写为可以序列化的类是实现Serializable接口，在Hadoop中将一个类写为可以序列化的类是实现Writable接口，它是一个最顶级的接口。

#### Hadoop对基本数据类型的包装

所有Java基本类型的可写包装器，除了char（可以是存储在IntWritable中）。所有的都有一个get（）和set（）方法来检索和存储包装值。　

![img](https://images2017.cnblogs.com/blog/999804/201710/999804-20171022173011896-1280163776.png)

#### Writable接口

Writable接口定义了两个方法：一个将其状态写到DataOutput二进制流，另一个从DataInput二进制流读取状态。

　　　　![img](https://images2017.cnblogs.com/blog/999804/201710/999804-20171022160813131-382979502.png)

``` java
public class MyWritable implements Writable {
    // Some data     
    private int counter;
    private long timestamp;

    public void write(DataOutput out) throws IOException {
        out.writeInt(counter);
        out.writeLong(timestamp);
    }

    public void readFields(DataInput in) throws IOException {
        counter = in.readInt();
        timestamp = in.readLong();
    }

    public static MyWritable read(DataInput in) throws IOException {
        MyWritable w = new MyWritable();
        w.readFields(in);
        return w;
    }
}
```

Writable的继承关系

　　![img](https://images2017.cnblogs.com/blog/999804/201710/999804-20171022174141302-1024250572.png)

#### WritableComparable<T>接口

#### RawComparator<T>接口

除了Comparator中继承的两个方法，它自己也定义了一个方法有6个参数，这是在字节流的层面上去做比较。（第一个参数：指定字节数组，第二个参数：从哪里开始比较，第三个参数：比较多长）

![img](https://images2017.cnblogs.com/blog/999804/201710/999804-20171022200534318-177034664.png)

## 基于文件的数据结构

### SequenceFile

SequenceFile是一个由二进制序列化过的key/value的字节流组成的文本存储文件，它可以在map/reduce过程中的input/output 的format时被使用。在map/reduce过程中，map处理文件的临时输出就是使用SequenceFile处理过的。 所以一般的SequenceFile均是在FileSystem中生成，供map调用的原始文件。

#### 特点

SequenceFile是append only的，于是你不能对已存在的key进行写操作。

SequenceFile可以作为小文件的容器，可以通过它将小文件包装起来传递给MapReduce进行处理。

SequenceFile提供了两种定位文件位置的方法

1. seek(long poisitiuion):poisition必须是记录的边界，否则调用next()方法时会报错
2. sync(long poisition):Poisition可以不是记录的边界，如果不是边界，会定位到下一个同步点，如果Poisition之后没有同步点了，会跳转到文件的结尾位置

#### 状态

SequenceFile 有三种压缩态：

1.   Uncompressed – 未进行压缩的状态
2.   record compressed - 对每一条记录的value值进行了压缩（文件头中包含上使用哪种压缩算法的信息）
3.   block compressed – 当数据量达到一定大小后，将停止写入进行整体压缩，整体压缩的方法是把所有的keylength,key,vlength,value 分别合在一起进行整体压缩，块的压缩效率要比记录的压缩效率高

#### 格式

每一个SequenceFile都包含一个“头”（header)。Header包含了以下几部分。

1. SEQ三个字母的byte数组
2. Version number的byte，目前为数字3的byte
3. Key和Value的类名
4. 压缩相关的信息
5. 其他用户定义的元数据
6. 同步标记，sync marker

对于每一条记录（K-V），其内部格式根据是否压缩而不同。SequenceFile的压缩方式有两种，“记录压缩”（record compression）和“块压缩”（block compression）。如果是记录压缩，则只压缩Value的值。如果是块压缩，则将多条记录一并压缩，包括Key和Value。具体格式如下面两图所示：

![img](https://img-blog.csdn.net/20141210223003754?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZXJpY19zdW5haA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![img](https://img-blog.csdn.net/20141210222919684?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZXJpY19zdW5haA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

#### 操作

``` java
public class SequenceFileDemo {
 
 private static final String OPERA_FILE = "./output.seq";
 /**
  * 随便从网上截取的一段文本
  */
 private static String[]    testArray = { "<plugin>                                                                     ",
         "  <groupId>org.apache.avro</groupId>                                         ",
         "  <artifactId>avro-maven-plugin</artifactId>                                 ",
         "  <version>1.7.7</version>                                                   ",
         "  <executions>                                                               ",
         "    <execution>                                                              ",
         "      <phase>generate-sources</phase>                                        ",
         "      <goals>                                                                ",
         "        <goal>schema</goal>                                                  ",
         "      </goals>                                                               ",
         "      <configuration>                                                        ",
         "        <sourceDirectory>${project.basedir}/src/main/avro/</sourceDirectory> ",
         "        <outputDirectory>${project.basedir}/src/main/java/</outputDirectory> ",
         "      </configuration>                                                       ",
         "    </execution>                                                             ",
         "  </executions>                                                              ",
         "</plugin>                                                                    ",
         "<plugin>                                                                     ",
         "  <groupId>org.apache.maven.plugins</groupId>                                ",
         "  <artifactId>maven-compiler-plugin</artifactId>                             ",
         "  <configuration>                                                            ",
         "    <source>1.6</source>                                                     ",
         "    <target>1.6</target>                                                     ",
         "  </configuration>                                                           ", "</plugin>"};
 
 public static void main(String[] args) throws IOException {
  writeSequenceFile(OPERA_FILE);
  readSequenceFile(OPERA_FILE);
 }
 
 private static void readSequenceFile(String inputFile) throws IOException {
  Configuration config = new Configuration();
  Path path = new Path(inputFile);
  SequenceFile.Reader reader = null;
  try {
   
   FileSystem fs = FileSystem.get(URI.create(inputFile), config);
   reader = new SequenceFile.Reader(fs, path, config);
   IntWritable key = new IntWritable();
   Text value = new Text();
   long posion = reader.getPosition();
   // reader.next()返回非空的话表示正在读，如果返回null表示已经读到文件结尾的地方
   while (reader.next(key, value)) {
    //打印同步点的位置信息
    String syncMark = reader.syncSeen() ? "*" : "";
    System.out.printf("[%s\t%s]\t%s\t%s\n", posion, syncMark, key, value);
    posion = reader.getPosition();
   }
  } finally {
   IOUtils.closeStream(reader);
  }
 
 }
 
 /**
  * 写Sequence File 文件
  *
  * @param outputFile
  * @throws IOException
  */
 private static void writeSequenceFile(String outputFile) throws IOException {
  Configuration config = new Configuration();
  Path path = new Path(outputFile);
  IntWritable key = new IntWritable();
  Text value = new Text();
  SequenceFile.Writer writer = null;
  try {
   
   FileSystem fs = FileSystem.get(URI.create(outputFile), config);
   writer = SequenceFile.createWriter(fs, config, path, key.getClass(), value.getClass());
   for (int i = 1; i < 2000; i++) {
    key.set(2000 - i);
    value.set(testArray[i % testArray.length]);
    System.out.printf("[%s]\t%s\t%s\n", writer.getLength() + "", key, value);
    // 通过Append方法进行写操作
    writer.append(key, value);
   }
  } finally {
   IOUtils.closeStream(writer);
  }
 
 }
}
```

### map file

MapFile是排序后的SequenceFile，MapFile由两部分组成分别是data和index。

index作为文件的数据索引，主要记录了每个Record的Key值，以及该Record在文件中的偏移位置。在MapFile被访问的时候，索引文件会被加载到内存，通过索引映射关系可以迅速定位到指定Record所在文件位置，因此，相对SequenceFile而言，MapFile的检索效率是最高的，缺点是会消耗一部分内存来存储index数据。

### 面向列

Hadoop中的文件格式大致上分为面向行和面向列两类：

- 面向行：同一行的数据存储在一起，即连续存储。SequenceFile,MapFile,Avro Datafile。采用这种方式，如果只需要访问行的一小部分数据，亦需要将整行读入内存，推迟序列化一定程度上可以缓解这个问题，但是从磁盘读取整行数据的开销却无法避免。面向行的存储适合于整行数据需要同时处理的情况。
- 面向列：整个文件被切割为若干列数据，每一列数据一起存储。Parquet , RCFile,ORCFile。面向列的格式使得读取数据时，可以跳过不需要的列，适合于只处于行的一小部分字段的情况。但是这种格式的读写需要更多的内存空间，因为需要缓存行在内存中（为了获取多行中的某一列）。同时不适合流式写入，因为一旦写入失败，当前文件无法恢复，而面向行的数据在写入失败时可以重新同步到最后一个同步点，所以Flume采用的是面向行的存储格式。

![logical-table.jpg-142.5kB](http://static.zybuluo.com/BrandonLin/eexzepzf0wuf29w2e7wllcja/logical-table.jpg)

![file.png-201kB](http://static.zybuluo.com/BrandonLin/b6q7z6ueqipvtrosz6gkzhi7/file.png)