---
title: HDFS介绍
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

# 3 Hdfs架构

## 3.1 概述

1. HDFS集群分为两大角色：NameNode、DataNode  (Secondary Namenode)
2. NameNode负责管理整个文件系统的元数据
3. DataNode 负责管理用户的文件数据块
4. 文件会按照固定的大小（blocksize）切成若干块后分布式存储在若干台datanode上
5. 每一个文件块可以有多个副本，并存放在不同的datanode上
6. Datanode会定期向Namenode汇报自身所保存的文件block信息，而namenode则会负责保持文件的副本数量
7. HDFS的内部工作机制对客户端保持透明，客户端请求访问HDFS都是通过向namenode申请来进行

##  3.2 Namenode 和 Datanode 

HDFS采用master/slave架构。一个HDFS集群是由一个Namenode和一定数目的Datanodes组成。

Namenode是一个中心服务器，负责管理文件系统的名字空间(namespace)以及客户端对文件的访问。

集群中的Datanode一般是一个节点一个，负责管理它所在节点上的存储。HDFS暴露了文件系统的名字空间，用户能够以文件的形式在上面存储数据。

从内部看，一个文件其实被分成一个或多个数据块，这些块存储在一组Datanode上。Namenode执行文件系统的名字空间操作，比如打开、关闭、重命名文件或目录。它也负责确定数据块到具体Datanode节点的映射。Datanode负责处理文件系统客户端的读写请求。在Namenode的统一调度下进行数据块的创建、删除和复制。

![HDFS Architecture](http://hadoop.apache.org/docs/r2.9.0/hadoop-project-dist/hadoop-hdfs/images/hdfsarchitecture.png)

集群中单一Namenode的结构大大简化了系统的架构。Namenode是所有HDFS元数据的仲裁者和管理者，这样，用户数据永远不会流过Namenode。

## 3.3 文件系统的命名空间（namespace）

HDFS支持传统的层次型文件组织结构。文件系统名字空间的层次结构和大多数现有的文件系统类似：用户可以创建、删除、移动或重命名文件和目录。

Namenode负责维护文件系统的名字空间，任何对文件系统名字空间或属性的修改都将被Namenode记录下来。应用程序可以设置HDFS保存的文件的副本数目。文件副本的数目称为文件的副本系数，这个信息也是由Namenode保存的。

## 3.4 数据复制 

HDFS被设计成能够在一个大集群中跨机器可靠地存储超大文件。它将每个文件存储成一系列的数据块，除了最后一个，所有的数据块都是同样大小的。为了容错，文件的所有数据块都会有副本。每个文件的数据块大小和副本系数都是可配置的。副本系数可以在文件创建的时候指定，也可以在之后改变。HDFS中的文件都是一次性写入的，并且严格要求在任何时候只能有一个写入者。

Namenode全权管理数据块的复制，它周期性地从集群中的每个datanode接收心跳信号和块状态报告(Blockreport)。接收到心跳信号意味着该Datanode节点工作正常。块状态报告包含了一个该Datanode上所有数据块的列表。

### 3.4.1 副本存放

副本的存放是HDFS可靠性和性能的关键。HDFS采用一种称为机架感知(rack-aware)的策略来改进数据的可靠性、可用性和网络带宽的利用率。

通过一个[机架感知](http://hadoop.apache.org/docs/r1.0.4/cn/cluster_setup.html#Hadoop%E7%9A%84%E6%9C%BA%E6%9E%B6%E6%84%9F%E7%9F%A5)的过程，Namenode可以确定每个Datanode所属的机架id。将副本存放在不同的机架上，有效防止当整个机架失效时数据的丢失，但是，因为这种策略的一个写操作需要传输数据块到多个机架，这增加了写的代价。 

在大多数情况下，副本系数是3，HDFS的存放策略是将一个副本存放在本地机架的节点上，一个副本放在同一机架的另一个节点上，最后一个副本放在不同机架的节点上。在这种策略下，副本并不是均匀分布在不同的机架上。三分之一的副本在一个节点上，三分之二的副本在一个机架上，其他副本均匀分布在剩下的机架中。如果replication factor大于3，则第4个以后的数据随机放在其他机架上，但保证每个机架副本数`<(replicas - 1) / racks + 2`

### 3.4.2 副本选择 

为了降低整体的带宽消耗和读取延时，HDFS会尽量让读取程序读取离它最近的副本。

### 3.4.3 安全模式 

Namenode启动后会进入一个称为安全模式的特殊状态。处于安全模式的Namenode是不会进行数据块的复制的。Namenode从所有的 
Datanode接收心跳信号和块状态报告。块状态报告包括了某个Datanode所有的数据块列表。每个数据块都有一个指定的最小副本数。当Namenode检测确认某个数据块的副本数目达到这个最小值，那么该数据块就会被认为是副本安全(safely replicated)的；在一定百分比（这个参数可配置）的数据块被Namenode检测确认是安全之后（加上一个额外的30秒等待时间），Namenode将退出安全模式状态。接下来它会确定还有哪些数据块的副本没有达到指定数目，并将这些数据块复制到其他Datanode上。

### 3.4.4 文件系统元数据的持久化

Namenode上保存着HDFS的名字空间。对于任何对文件系统元数据产生修改的操作，Namenode都会使用一种称为EditLog的事务日志记录下来。Namenode在本地操作系统的文件系统中存储这个Editlog。整个文件系统的名字空间，包括数据块到文件的映射、文件的属性等，都存储在一个称为FsImage的文件中，这个文件也是放在Namenode所在的本地文件系统上。

Namenode在内存中保存着整个文件系统的名字空间和文件数据块映射(Blockmap)的映像。这个关键的元数据结构设计得很紧凑，因而一个有4G内存的Namenode足够支撑大量的文件和目录。当Namenode启动时或者到达checkpoint 的条件时，它从硬盘中读取Editlog和FsImage，将所有Editlog中的事务作用在内存中的FsImage上，并将这个新版本的FsImage从内存中保存到本地磁盘上，然后删除旧的Editlog。这个过程是checkpoint

```properties
dfs.namenode.checkpoint.period
dfs.namenode.checkpoint.txns
```

Datanode将HDFS数据以文件的形式存储在本地的文件系统中，它并不知道有关HDFS文件的信息。它把每个HDFS数据块存储在本地文件系统的一个单独的文件中。Datanode并不在同一个目录创建所有的文件，实际上，它用试探的方法来确定每个目录的最佳文件数目，并且在适当的时候创建子目录。在同一个目录中创建所有的本地文件并不是最优的选择，这是因为本地文件系统可能无法高效地在单个目录中支持大量的文件。当一个Datanode启动时，它会扫描本地文件系统，产生一个这些本地文件对应的所有HDFS数据块的列表，然后作为报告发送到Namenode，这个报告就是块状态报告。

## 3.5 通信协议

所有的HDFS通讯协议都是建立在TCP/IP协议之上。客户端通过一个可配置的TCP端口连接到Namenode，通过**ClientProtocol**协议与**Namenode**交互。而Datanode使用DatanodeProtocol协议与Namenode交互。一个远程过程调用(RPC)模型被抽象出来封装ClientProtocol和Datanodeprotocol协议。在设计上，Namenode不会主动发起RPC，而是响应来自客户端或 Datanode 的RPC请求。 

## 3.6 健壮性 

HDFS的主要目标就是即使在出错的情况下也要保证数据存储的可靠性。常见的三种出错情况是：Namenode出错, Datanode出错和网络隔离(network partitions)。

### 3.6.1 磁盘数据错误，心跳检测和重新复制 

每个Datanode节点周期性地向Namenode发送心跳信号。网络割裂可能导致一部分Datanode跟Namenode失去联系。Namenode通过心跳信号的缺失来检测这一情况，并将这些近期不再发送心跳信号Datanode标记为宕机，不会再将新的IO请求发给它们。任何存储在宕机Datanode上的数据将不再有效。Datanode的宕机可能会引起一些数据块的副本系数低于指定值，Namenode不断地检测这些需要复制的数据块，一旦发现就启动复制操作。在下列情况下，可能需要重新复制：某个Datanode节点失效，某个副本遭到损坏，Datanode上的硬盘错误，或者文件的副本系数增大。

### 3.6.2 群集的负载均衡

HDFS的架构支持数据均衡策略。如果某个Datanode节点上的空闲空间低于特定的临界点，按照均衡策略系统就会自动地将数据从这个Datanode移动到其他空闲的Datanode。当对某个文件的请求突然增加，那么也可能启动一个计划创建该文件新的副本，并且同时重新平衡集群中的其他数据。这些均衡策略目前还没有实现。

### 3.6.3 数据完整性

checksum。从某个Datanode获取的数据块有可能是损坏的，可能是由Datanode的存储设备错误、网络错误或者软件bug造成的。HDFS客户端软件实现了对HDFS文件内容的校验和(checksum)检查。当客户端创建一个新的HDFS文件，会计算这个文件每个数据块的校验和，并将校验和作为一个单独的隐藏文件保存在同一个HDFS名字空间下。当客户端获取文件内容后，它会检验从Datanode获取的数据跟相应的校验和文件中的校验和是否匹配，如果不匹配，客户端可以选择从其他Datanode获取该数据块的副本。

### 3.6.4元数据磁盘错误         

FsImage和Editlog是HDFS的核心数据结构。Namenode可以配置成支持维护多个FsImage和Editlog的副本。任何对FsImage或者Editlog的修改，都将同步到它们的副本上。这可能会降低Namenode每秒处理事务数量。然而这个代价是可以接受的，因为即使HDFS的应用是数据密集的，它们也非元数据密集的。

### 3.6.5 快照

回滚。快照支持某一特定时刻的数据的复制备份。利用快照，可以让HDFS在数据损坏时恢复到过去一个已知正确的时间点。

## 3.7 数据组织 

## 3.7.1 数据块

HDFS被设计成支持大文件，适用需要处理大规模的数据集的应用。HDFS支持文件的“一次写入多次读取”语义。一个典型的数据块大小是128MB。因而，HDFS中的文件总是按照128M被切分成不同的块，每个块尽可能地存储于不同的Datanode中。

> 小于块大小的小文件不会占用整个HDFS块空间。也就是说，较多的小文件会占用更多的NAMENODE的内存（记录了文件的位置等信息）

### 3.7.2 Staging

客户端创建文件的请求其实并没有立即发送给Namenode，会先将文件数据缓存到本地的一个临时文件。

1. 先将文件数据缓存到本地的一个临时文件
2. 当这个临时文件累积的数据量超过一个数据块的大小，客户端才会联系Namenode。Namenode将文件名插入文件系统的层次结构中，并返回Datanode的标识符和目标数据块给客户端。
3. 客户端将这块数据从本地临时文件上传到指定的Datanode上
4. 客户端告诉Namenode文件已经关闭。此时Namenode才将文件创建操作提交到日志里进行存储。如果Namenode在文件关闭前宕机了，则该文件将丢失

### 3.7.3 流水线复制 

当客户端向HDFS文件写入数据的时候，一开始是写到本地临时文件中。当本地临时文件累积到一个数据块的大小时，客户端会从Namenode获取一个Datanode列表用于存放副本。然后客户端开始向第一个Datanode传输数据，第一个Datanode一小部分一小部分(4KB)地接收数据，将每一部分写入本地仓库，并同时传输该部分到列表中第二个Datanode节点。第二个Datanode也是这样，一小部分一小部分地接收数据，写入本地仓库，并同时传给第三个Datanode。最后，第三个Datanode接收数据并存储在本地。因此，Datanode能流水线式地从前一个节点接收数据，并在同时转发给下一个节点，数据以流水线的方式从前一个Datanode复制到下一个。

## 3.8 存储空间回收 

### 3.8.1 文件的删除和恢复 

当用户或应用程序删除某个文件时，这个文件并没有立刻从HDFS中删除。实际上，HDFS会将这个文件重命名转移到/trash目录。只要文件还在/trash目录中，该文件就可以被迅速地恢复。文件在/trash中保存的时间是可配置的

### 3.8.2 减少副本系数 

当一个文件的副本系数被减小后，Namenode会选择过剩的副本删除。下次心跳检测时会将该信息传递给Datanode。Datanode遂即移除相应的数据块，集群中的空闲空间加大。

## 3.9 HDFS写文件流程

客户端要向HDFS写数据，首先要跟namenode通信以确认可以写文件并获得接收文件block的datanode，然后，客户端按顺序将文件逐个block传递给相应datanode，并由接收到block的datanode负责向其他datanode复制block的副本

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fm79ft3j9jj30ox0fxgmj.jpg)

1. 根namenode通信请求上传文件，namenode检查目标文件是否已存在，父目录是否存在

2. namenode返回是否可以上传

3. client先缓存本地第一个block，完成后请求第一个 block该传输到哪些datanode服务器上

4. namenode返回3个datanode服务器ABC

5. 建立管道，client请求3台dn中的一台A上传数据（本质上是一个RPC调用，建立pipeline），A收到请求会继续调用B，然后B调用C，将真个pipeline建立完成，逐级返回客户端

6. client开始往A上传第一个block（先从磁盘读取数据放到一个本地内存缓存），以packet为单位，A收到一个packet就会传给B，B传给C；A每传一个packet会放入一个应答队列等待应答

7. 当一个block传输完成之后，client再次请求namenode上传第二个block的服务器。

   ![](http://7xsvi4.com2.z0.glb.clouddn.com/hdfs%20write%20data.png)

## 3.10 HDFS写文件流程

客户端将要读取的文件路径发送给namenode，namenode获取文件的元信息（主要是block的存放位置信息）返回给客户端，客户端根据返回的信息找到相应datanode逐个获取文件的block并在客户端本地进行数据追加合并从而获得整个文件

![](http://ww1.sinaimg.cn/large/0063bT3gly1fm79qf5bfvj30se0dsjrw.jpg)

1. 跟namenode通信查询元数据，找到文件块所在的datanode服务器
2. 挑选一台datanode（就近原则，然后随机）服务器，请求建立socket流
3. datanode开始发送数据（从磁盘里面读取数据放入流，以packet为单位来做校验）
4. 客户端以packet为单位接收，现在本地缓存，然后写入目标文件

## 3.11 NAMENODE工作机制

学习目标：理解namenode的工作机制尤其是**元数据管理**机制，以增强对HDFS工作原理的理解，及培养hadoop集群运营中“性能调优”、“namenode”故障问题的分析解决能力

问题场景：

1. 集群启动后，可以查看文件，但是上传文件时报错，打开web页面可看到namenode正处于safemode状态，怎么处理？
2. Namenode服务器的磁盘故障导致namenode**宕机，如何挽救集群及数据？
3. Namenode是否可以有多个？namenode内存要配置多大？namenode跟集群数据存储能力有关系吗？
4. 文件的blocksize究竟调大好还是调小好？

*……*

诸如此类问题的回答，都需要基于对namenode自身的工作原理的深刻理解

### 3.11.1  NAMENODE职责

NAMENODE职责：

- 负责客户端请求的响应
- 元数据的管理（查询，修改）

### 3.11.2 元数据管理

namenode对数据的管理采用了三种存储形式：

- 内存元数据(NameSystem)
- 磁盘元数据镜像文件FsImage
- 数据操作日志文件Editlog（可通过日志运算出元数据）

### 3.11.3 查看FsImage和Editlog状态

Offline Edits Viewer is a tool to parse the Edits log file.

```shell
bash$ bin/hdfs oev -p xml -i edits -o edits.xml
bash$ bin/hdfs oev -i edits -o edits.xml
```

### 3.11.4 元数据的checkpoint

每隔一段时间，会由secondary namenode将namenode上积累的所有edits和一个最新的fsimage下载到本地，并加载到内存进行merge（这个过程称为checkpoint）

**checkpoint的详细过程**

SecondaryNameNode有两个作用，一是镜像备份，二是日志与镜像的定期合并。

![](http://www.processon.com/chart_image/535371590cf2bb589c5e2391.png)

**checkpoint操作的触发条件配置参数**

```properties
dfs.namenode.checkpoint.check.period=60  #检查触发条件是否满足的频率，60秒
dfs.namenode.checkpoint.dir=file://${hadoop.tmp.dir}/dfs/namesecondary
#以上两个参数做checkpoint操作时，secondary namenode的本地工作目录
dfs.namenode.checkpoint.edits.dir=${dfs.namenode.checkpoint.dir}

dfs.namenode.checkpoint.max-retries=3  #最大重试次数
dfs.namenode.checkpoint.period=3600  #两次checkpoint之间的时间间隔3600秒
dfs.namenode.checkpoint.txns=1000000 #两次checkpoint之间最大的操作记录

```



### 3.11.5 元数据目录说明

在第一次部署好Hadoop集群的时候，我们需要在NameNode（NN）节点上格式化磁盘：

```shell
$HADOOP_HOME/bin/hdfs namenode -format
```

格式化完成之后，将会在$dfs.namenode.name.dir/current目录下如下的文件结构

```shell
current/
|-- VERSION
|-- edits_*
|-- fsimage_0000000000008547077
|-- fsimage_0000000000008547077.md5
`-- seen_txid
```

其中的dfs.name.dir是在hdfs-site.xml文件中配置的，默认值file://\${hadoop.tmp.dir}/dfs/name,  hadoop.tmp.dir是在core-site.xml中配置的，默认值/tmp/hadoop-\${user.name}

dfs.namenode.name.dir属性可以配置多个目录，如/data1/dfs/name,/data2/dfs/name,/data3/dfs/name,....。各个目录存储的文件结构和内容都完全一样，相当于备份，这样做的好处是当其中一个目录损坏了，也不会影响到Hadoop的元数据，特别是当其中一个目录是NFS（网络文件系统Network File System，NFS）之上，即使你这台机器损坏了，元数据也得到保存。

**$dfs.namenode.name.dir/current/目录**

1. VERSION文件是Java属性文件，内容大致如下：

   ```properties
   #Wed Dec 06 08:57:50 CST 2017
   namespaceID=1084867803
   clusterID=CID-2489a7d6-7f2e-4088-8f29-349ce1f5275f
   cTime=1511719134838
   storageType=NAME_NODE
   blockpoolID=BP-1249732924-192.168.2.32-1511719134838
   layoutVersion=-63
   ```

   - namespaceID是文件系统的唯一标识符，在文件系统首次格式化之后生成的；
   - storageType说明这个文件存储的是什么进程的数据结构信息（如果是DataNode，storageType=DATA_NODE）；
   - cTime表示NameNode存储时间的创建时间，由于我的NameNode没有更新过，所以这里的记录值为0，以后对NameNode升级之后，cTime将会记录更新时间戳；
   - layoutVersion表示HDFS永久性数据结构的版本信息， 只要数据结构变更，版本号也要递减，此时的HDFS也需要升级，否则磁盘仍旧是使用旧版本的数据结构，这会导致新版本的NameNode无法使用；
   - clusterID是系统生成或手动指定的集群ID，在-clusterid选项中可以使用它；
   - blockpoolID：是针对每一个Namespace所对应的blockpool的ID

2. seen_txid非常重要，是存放transactionId的文件。format之后是0，它代表的是namenode里面的edits\_\*文件的尾数，namenode重启的时候，会按照seen_txid的数字，循序从头跑edits_0000001~到seen_txid的数字。所以当你的hdfs发生异常重启的时候，一定要比对seen_txid内的数字是不是你edits最后的尾数，不然会发生建置namenode时metaData的资料有缺少，导致误删Datanode上多余Block的资讯。

3. $dfs.namenode.name.dir/current目录下在format的同时也会生成fsimage和edits文件，及其对应的md5校验文件。

## 3.12 DATANODE的工作机制

问题场景：

1. 集群容量不够，怎么扩容？
2. 如果有一些datanode宕机，该怎么办？
3. datanode明明已启动，但是集群中的可用datanode列表中就是没有，怎么办？

Datanode工作职责:

1. 存储管理用户的文件块数据

2. 定期向namenode汇报自身所持有的block信息（通过心跳信息上报）

   > 这点很重要，因为，当集群中发生某些block副本失效时，集群如何恢复block初始副本数量的问题
   >
   > ``` xml
   > <property>
   > 	<name>dfs.blockreport.intervalMsec</name>
   > 	<value>3600000</value>
   > 	<description>Determines block reporting interval in milliseconds.</description>
   > </property>
   > ```

**Datanode掉线判断时限参数**

​	datanode进程死亡或者网络故障造成datanode无法与namenode通信，namenode不会立即把该节点判定为死亡，要经过一段时间，这段时间暂称作超时时长。HDFS默认的超时时长为10分钟+30秒。如果定义超时时间为timeout，则超时时长的计算公式为：

​	timeout  = 2 * heartbeat.recheck.interval + 10 * dfs.heartbeat.interval

默认的heartbeat.recheck.interval 大小为5分钟，dfs.heartbeat.interval默认为3秒。hdfs-site.xml 配置文件中的heartbeat.recheck.interval的单位为毫秒，dfs.heartbeat.interval的单位为秒

# 4 hdfs java开发

依赖：

``` xml
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-client</artifactId>
    <version>2.9.0</version>
</dependency>

```

## 4.1 获取api中的客户端对象

``` java
Configuration conf = new Configuration()
FileSystem fs = FileSystem.get(conf)

```

我们的操作目标是HDFS，所以获取到的fs对象应该是DistributedFileSystem的实例。get方法是从conf中的一个参数 fs.defaultFS的配置值判断用哪种文件系统。如果我们的代码中没有指定fs.defaultFS，并且工程classpath下也没有给定相应的配置，conf中的默认值就来自于hadoop的jar包中的core-default.xml，默认值为： file:///，则获取的是一个本地文件系统的客户端对象

## 4.2 DistributedFileSystem实例对象所具备的方法

![](http://ww1.sinaimg.cn/large/0063bT3ggy1fm7iie3w5bj30lz0lkwgn.jpg)

## 4.3 HDFS客户端操作数据代码示例

文件操作

``` java
public class HdfsClient {

	FileSystem fs = null;

	@Before
	public void init() throws Exception {

		// 构造一个配置参数对象，设置一个参数：我们要访问的hdfs的URI
		// 从而FileSystem.get()方法就知道应该是去构造一个访问hdfs文件系统的客户端，以及hdfs的访问地址
		// new Configuration();的时候，它就会去加载jar包中的hdfs-default.xml
		// 然后再加载classpath下的hdfs-site.xml
		Configuration conf = new Configuration();
		conf.set("fs.defaultFS", "hdfs://hdp-node01:9000");
		/**
		 * 参数优先级： 1、客户端代码中设置的值 2、classpath下的用户自定义配置文件 3、然后是服务器的默认配置
		 */
		conf.set("dfs.replication", "3");

		// 获取一个hdfs的访问客户端，根据参数，这个实例应该是DistributedFileSystem的实例
		// fs = FileSystem.get(conf);

		// 如果这样去获取，那conf里面就可以不要配"fs.defaultFS"参数，而且，这个客户端的身份标识已经是hadoop用户
		fs = FileSystem.get(new URI("hdfs://hdp-node01:9000"), conf, "hadoop");

	}

	/**
	 * 往hdfs上传文件
	 * 
	 * @throws Exception
	 */
	@Test
	public void testAddFileToHdfs() throws Exception {

		// 要上传的文件所在的本地路径
		Path src = new Path("g:/redis-recommend.zip");
		// 要上传到hdfs的目标路径
		Path dst = new Path("/aaa");
		fs.copyFromLocalFile(src, dst);
		fs.close();
	}

	/**
	 * 从hdfs中复制文件到本地文件系统
	 */
	@Test
	public void testDownloadFileToLocal() throws IllegalArgumentException, IOException {
		fs.copyToLocalFile(new Path("/jdk-7u65-linux-i586.tar.gz"), new Path("d:/"));
		fs.close();
	}

	@Test
	public void testMkdirAndDeleteAndRename() throws IllegalArgumentException, IOException {
		// 创建目录
		fs.mkdirs(new Path("/a1/b1/c1"));
		// 删除文件夹 ，如果是非空文件夹，参数2必须给值true
		fs.delete(new Path("/aaa"), true);
		// 重命名文件或文件夹
		fs.rename(new Path("/a1"), new Path("/a2"));

	}

	/**
	 * 查看目录信息，只显示文件
	 */
	@Test
	public void testListFiles() throws FileNotFoundException, IllegalArgumentException, IOException {
		// 思考：为什么返回迭代器，而不是List之类的容器
		RemoteIterator<LocatedFileStatus> listFiles = fs.listFiles(new Path("/"), true);
		while (listFiles.hasNext()) {
			LocatedFileStatus fileStatus = listFiles.next();
			System.out.println(fileStatus.getPath().getName());
			System.out.println(fileStatus.getBlockSize());
			System.out.println(fileStatus.getPermission());
			System.out.println(fileStatus.getLen());
			BlockLocation[] blockLocations = fileStatus.getBlockLocations();
			for (BlockLocation bl : blockLocations) {
				System.out.println("block-length:" + bl.getLength() + "--" + "block-offset:" + bl.getOffset());
				String[] hosts = bl.getHosts();
				for (String host : hosts) {
					System.out.println(host);
				}
			}
			System.out.println("--------------为angelababy打印的分割线--------------");
		}
	}

	/**
	 * 查看文件及文件夹信息
	 */
	@Test
	public void testListAll() throws FileNotFoundException, IllegalArgumentException, IOException {

		FileStatus[] listStatus = fs.listStatus(new Path("/"));

		String flag = "d--             ";
		for (FileStatus fstatus : listStatus) {
			if (fstatus.isFile())  flag = "f--         ";
			System.out.println(flag + fstatus.getPath().getName());
		}
	}
}

```

通过流的方式访问hdfs

``` java
/**
 * 相对那些封装好的方法而言的更底层一些的操作方式
 * 上层那些mapreduce   spark等运算框架，去hdfs中获取数据的时候，就是调的这种底层的api
 * @author
 *
 */
public class StreamAccess {
	
	FileSystem fs = null;

	@Before
	public void init() throws Exception {
		Configuration conf = new Configuration();
		fs = FileSystem.get(new URI("hdfs://hdp-node01:9000"), conf, "hadoop");
	}
	
	/**
	 * 通过流的方式上传文件到hdfs
	 */
	@Test
	public void testUpload() throws Exception {		
		FSDataOutputStream outputStream = fs.create(new Path("/angelababy.love"), true);
		FileInputStream inputStream = new FileInputStream("c:/angelababy.love");	
		IOUtils.copy(inputStream, outputStream);		
	}
	
	@Test
	public void testDownLoadFileToLocal() throws IllegalArgumentException, IOException{
		//先获取一个文件的输入流----针对hdfs上的
		FSDataInputStream in = fs.open(new Path("/jdk-7u65-linux-i586.tar.gz"));		
		//再构造一个文件的输出流----针对本地的
		FileOutputStream out = new FileOutputStream(new File("c:/jdk.tar.gz"));	
		//再将输入流中数据传输到输出流
		IOUtils.copyBytes(in, out, 4096);			
	}	
	
	/**
	 * hdfs支持随机定位进行文件读取，而且可以方便地读取指定长度
	 * 用于上层分布式运算框架并发处理数据
	 */
	@Test
	public void testRandomAccess() throws IllegalArgumentException, IOException{
		//先获取一个文件的输入流----针对hdfs上的
		FSDataInputStream in = fs.open(new Path("/iloveyou.txt"));
		//可以将流的起始偏移量进行自定义
		in.seek(22);	
		//再构造一个文件的输出流----针对本地的
		FileOutputStream out = new FileOutputStream(new File("c:/iloveyou.line.2.txt"));
		IOUtils.copyBytes(in,out,19L,true);		
	}
	
	/**
	 * 显示hdfs上文件的内容
	 */
	@Test
	public void testCat() throws IllegalArgumentException, IOException{	
		FSDataInputStream in = fs.open(new Path("/iloveyou.txt"));	
		IOUtils.copyBytes(in, System.out, 1024);
	}
}

```

获取一个文件的所有block位置信息，然后读取指定block中的内容

``` java
@Test
	public void testCat() throws IllegalArgumentException, IOException{
		
		FSDataInputStream in = fs.open(new Path("/weblog/input/access.log.10"));
		//拿到文件信息
		FileStatus[] listStatus = fs.listStatus(new Path("/weblog/input/access.log.10"));
		//获取这个文件的所有block的信息
		BlockLocation[] fileBlockLocations = fs.getFileBlockLocations(listStatus[0], 0L, listStatus[0].getLen());
		//第一个block的长度
		long length = fileBlockLocations[0].getLength();
		//第一个block的起始偏移量
		long offset = fileBlockLocations[0].getOffset();
		
		System.out.println(length);
		System.out.println(offset);
		
		//获取第一个block写入输出流
//		IOUtils.copyBytes(in, System.out, (int)length);
		byte[] b = new byte[4096];
		
		FileOutputStream os = new FileOutputStream(new File("d:/block0"));
		while(in.read(offset, b, 0, 4096)!=-1){
			os.write(b);
			offset += 4096;
			if(offset>=length) return;
		};
		os.flush();
		os.close();
		in.close();
	}

```





