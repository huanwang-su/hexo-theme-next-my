---
title: MapReduce深入
date: 2018/3/15 20:46:25
category:
- 大数据
- MapReduce
tag:
- MapReduce
comments: true  
---
# 1 MapReduce原理

mapreduce&yarn的工作机制

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fml5yxee4jj31ty0n442c.jpg)

mapReduce原理剖析

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fml5yxafipj31r80h0mzc.jpg)

Shuffle原理

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fml5yxiyb3j31ty0n4whw.jpg)

mapreduce运行全流程

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fml5yx8veaj311c0kgmyp.jpg)

mapTask任务分配机制

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fml5yxdrjwj313s1dg76u.jpg)



# 2 疑问

1. map的结果输出在哪里，在hdfs的指定目录下面？如果是，那么目录结构又是什么样子的？又是如何指定的呢？

2. shuffle的过程是将所有的map结果进行归并和划分，哪里取数据，参考问题1，这个归并过程在reduce的进程里面处理，还是在reduce程序启动前已经处理好了。

3. 如何启动多个map任务，分片？那么如何分，代码都是一样的怎么看出来分了片。在map函数外部分？？inputformat？

   自定义Partitioner

   ```java
   @Override
   	public int getPartition(Text key, FlowBean value, int numPartitions) {
   		
   		String prefix = key.toString().substring(0,3);
   		Integer partNum = pmap.get(prefix);
   		
   		return (partNum==null?4:partNum);
   	}

   ```

   ​

4. map的输出会进行分区，分区是如何做的？分区的数量如何判断？分区和reduce的对应关系又是如何？

   自定义partition后，要根据自定义partitioner的逻辑设置相应数量的reduce task

   ```
   job.setNumReduceTasks(5);
   ```

   注意：

   - 如果reduceTask的数量>= getPartition的结果数  ，则会多产生几个空的输出文件part-r-000xx
   - 如果 1<reduceTask的数量<getPartition的结果数 ，则有一部分分区数据无处安放，会Exception！！！
   - 如果 reduceTask的数量=1，则不管mapTask端输出多少个分区文件，最终结果都交给这一个reduceTask，最终也就只会产生一个结果文件 part-r-00000

   ​