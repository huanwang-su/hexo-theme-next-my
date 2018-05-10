---
title: HDFS介绍和命令
date: 2018/3/15 20:46:25
category:
- 大数据
- hdfs
tag:
- hdfs
comments: true  
---

# 1 HDFS基本概念篇

Hadoop分布式文件系统(HDFS)被设计成适合运行在通用硬件(commodity hardware)上的分布式文件系统。它和现有的分布式文件系统有很多共同点。但同时，它和其他的分布式文件系统的区别也是很明显的。HDFS是一个高度容错性的系统，适合部署在廉价的机器上。HDFS能提供高吞吐量的数据访问，非常适合大规模数据集上的应用。

## 1.1 前沿

- 设计思想

  分而治之：将大文件、大批量文件，分布式存放在大量服务器上，以便于采取分而治之的方式对海量数据进行运算分析；

- 在大数据系统中作用：

  为各类分布式运算框架（如：mapreduce，spark，tez，……）提供数据存储服务

- 重点概念：文件切块，副本存放，元数据

## 1.2 HDFS的概念和特性

**首先，它是一个文件系统**，用于存储文件，通过统一的命名空间——目录树来定位文件

**其次，它是分布式的**，由很多服务器联合起来实现其功能，集群中的服务器有各自的角色；

**重要特性如下：**

1. HDFS中的文件在物理上是**分块存储（block）**，块的大小可以通过配置参数( dfs.blocksize)来规定，默认大小在hadoop2.x版本中是128M，老版本中是64M

2. HDFS文件系统会给客户端提供一个**统一的抽象目录树**，客户端通过路径来访问文件，形如：hdfs://namenode:port/dir-a/dir-b/dir-c/file.data

3. **目录结构及文件分块信息(元数据)**的管理由namenode节点承担。namenode是HDFS集群主节点，负责维护整个hdfs文件系统的目录树，以及每一个路径（文件）所对应的block块信息（block的id，及所在的datanode服务器）

4. 文件的各个block的存储管理由datanode节点承担

   datanode是HDFS集群从节点，每一个block都可以在多个datanode上存储多个副本（副本数量也可以通过参数设置dfs.replication）

5. （1）HDFS是设计成适应一次写入，多次读出的场景，且不支持文件的修改，支持追加

   > 注：适合用来做数据分析，并不适合用来做网盘应用，因为，不便修改，延迟大，网络开销大，成本太高

## 1.3 前提和设计目标 

### 硬件错误 

​	硬件错误是常态而不是异常。HDFS可能由成百上千的服务器所构成，每个服务器上存储着文件系统的部分数据。任一组件都有可能失效，因此错误检测和快速、自动的恢复是HDFS最核心的架构目标。       

### 流式数据访问 

​	运行在HDFS上的应用需要流式访问它们的数据集。HDFS的设计中更多的考虑到了数据批处理，而不是用户交互处理。比之数据访问的低延迟问题，更关键的在于数据访问的高吞吐量。            

### 大规模数据集 

​        运行在HDFS上的应用具有很大的数据集。HDFS上的一个典型文件大小一般都在G字节至T字节。因此，HDFS被调节以支持大文件存储。它应该能提供整体上高的数据传输带宽，能在一个集群里扩展到数百个节点。一个单一的HDFS实例应该能支撑数以千万计的文件。        

### 简单的一致性模型 

​        HDFS应用需要一个“一次写入多次读取”的文件访问模型。一个文件经过创建、写入和关闭之后就不需要改变。这一假设简化了数据一致性问题，并且使高吞吐量的数据访问成为可能。Map/Reduce应用或者网络爬虫应用都非常适合这个模型。目前扩充这个模型，使之支持文件的附加写操作。         

### “移动计算比移动数据更划算” 

​        一个应用请求的计算，离它操作的数据越近就越高效，在数据达到海量级别的时候更是如此。HDFS为应用提供了将它们自己移动到数据附近的接口。         

### 异构软硬件平台间的可移植性 

​        HDFS在设计的时候就考虑到平台的可移植性。这种特性方便了HDFS作为大规模数据应用平台的推广。

# 2 HDFS基本操作篇

## 2.1 HDFS的shell(命令行客户端)操作

### 2.1.1 支持的命令参数

```shell
[-appendToFile <localsrc> ... <dst>]
[-cat [-ignoreCrc] <src> ...]
[-checksum <src> ...]
[-chgrp [-R] GROUP PATH...]
[-chmod [-R] <MODE[,MODE]... | OCTALMODE> PATH...]
[-chown [-R] [OWNER][:[GROUP]] PATH...]
[-copyFromLocal [-f] [-p] <localsrc> ... <dst>]
[-copyToLocal [-p] [-ignoreCrc] [-crc] <src> ... <localdst>]
[-count [-q] <path> ...]
[-cp [-f] [-p] <src> ... <dst>]
[-createSnapshot <snapshotDir> [<snapshotName>]]
[-deleteSnapshot <snapshotDir> <snapshotName>]
[-df [-h] [<path> ...]]
[-du [-s] [-h] <path> ...]
[-expunge]
[-get [-p] [-ignoreCrc] [-crc] <src> ... <localdst>]
[-getfacl [-R] <path>]
[-getmerge [-nl] <src> <localdst>]
[-help [cmd ...]]
[-ls [-d] [-h] [-R] [<path> ...]]
[-mkdir [-p] <path> ...]
[-moveFromLocal <localsrc> ... <dst>]
[-moveToLocal <src> <localdst>]
[-mv <src> ... <dst>]
[-put [-f] [-p] <localsrc> ... <dst>]
[-renameSnapshot <snapshotDir> <oldName> <newName>]
[-rm [-f] [-r|-R] [-skipTrash] <src> ...]
[-rmdir [--ignore-fail-on-non-empty] <dir> ...]
[-setfacl [-R] [{-b|-k} {-m|-x <acl_spec>} <path>]|[--set <acl_spec> <path>]]
[-setrep [-R] [-w] <rep> <path> ...]
[-stat [format] <path> ...]
[-tail [-f] <file>]
[-test -[defsz] <path>]
[-text [-ignoreCrc] <src> ...]
[-touchz <path> ...]
[-usage [cmd ...]]

```

### 2.1.2 常用命令参数介绍

| -help               功能：输出这个命令参数手册        |
| ---------------------------------------- |
| **-ls                  **  **功能：显示目录信息**  *示例： hadoop fs -ls hdfs://hadoop-server01:9000/*  备注：这些参数中，所有的hdfs路径都可以简写  -->hadoop fs -ls /    等同于上一条命令的效果 |
| **-mkdir              **  **功能：在hdfs**上创建目录  *示例：hadoop fs   -mkdir  -p  /aaa/bbb/cc/dd* |
| **-moveFromLocal          功能：从本地剪切粘贴到hdfs**  *示例：hadoop  fs  -  moveFromLocal  /home/hadoop/a.txt  /aaa/bbb/cc/dd* |
| **-moveToLocal              **  **功能：从hdfs**剪切粘贴到本地  *示例：hadoop  fs  -  moveToLocal   /aaa/bbb/cc/dd  /home/hadoop/a.txt* |
| **--appendToFile  **  **功能：追加一个文件到已经存在的文件末尾**  *示例：hadoop  fs  -appendToFile  ./hello.txt   hdfs://hadoop-server01:9000/hello.txt*  *可以简写为：*  *Hadoop  fs  -appendToFile  ./hello.txt   /hello.txt* |
| **-cat  **                      **功能：显示文件内容   **  *示例：hadoop fs -cat   /hello.txt* |
| **-tail                     **  **功能：显示一个文件的末尾**  *示例：hadoop  fs   -tail  /weblog/access_log.1* |
| **-text                  **  **功能：以字符形式打印一个文件的内容**  *示例：hadoop  fs   -text  /weblog/access_log.1* |
| **-chgrp **  **-chmod**  **-chown**  **功能：linux** **文件系统中的用法一样，对文件所属权限**  *示例：*  *hadoop  fs  -chmod   666  /hello.txt*  *hadoop  fs  -chown   someuser:somegrp   /hello.txt* |
| **-copyFromLocal    **  **功能：从本地文件系统中拷贝文件到hdfs路径去**  *示例：hadoop  fs  -copyFromLocal  ./jdk.tar.gz  /aaa/* |
| **-copyToLocal      **  **功能：从hdfs拷贝到本地**  *示例：hadoop fs -copyToLocal  /aaa/jdk.tar.gz* |
| **-cp              **  **功能：从hdfs的一个路径拷贝hdfs的另一个路径**  *示例： hadoop  fs  -cp   /aaa/jdk.tar.gz  /bbb/jdk.tar.gz.2* |
| **-mv                     **  **功能：在hdfs目录中移动文件**  *示例： hadoop  fs  -mv  /aaa/jdk.tar.gz  /* |
| **-get              **  **功能：等同于copyToLocal****，就是从hdfs****下载文件到本地**  示例：hadoop fs -get  /aaa/jdk.tar.gz |
| **-getmerge             **  **功能：合并下载多个文件**  *示例：**比如hdfs**的目录 /aaa/**下有多个文件:log.1, log.2,log.3,...*  hadoop  fs -getmerge /aaa/log.* ./log.sum |
| **-put                **  **功能：等同于copyFromLocal**  *示例：hadoop  fs  -put  /aaa/jdk.tar.gz  /bbb/jdk.tar.gz.2* |
| **-rm                **  **功能：删除文件或文件夹**  *示例：hadoop fs -rm -r /aaa/bbb/* |
| **-rmdir                 **  **功能：删除空目录**  *示例：hadoop  fs   -rmdir   /aaa/bbb/ccc* |
| **-df               **  **功能：统计文件系统的可用空间信息**  *示例：hadoop  fs  -df   -h  /* |
| **-du **           **功能：统计文件夹的大小信息**  *示例：*  *hadoop  fs  -du   -s  -h /aaa/\** |
| **-count         **  **功能：统计一个指定目录下的文件节点数量**  *示例：hadoop fs -count /aaa/* |
| **-setrep                **  **功能：设置hdfs中文件的副本数量**  *示例：hadoop fs -setrep 3 /aaa/jdk.tar.gz* |

