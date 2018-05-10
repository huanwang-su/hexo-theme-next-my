---
title: HDFS架构
date: 2018/3/15 20:46:25
category:
- 大数据
- hdfs
tag:
- hdfs
comments: true  
---

# 3 Hdfs架构

设计：

1. 超大文件
2. 流式数据访问
3. 运行在廉价商用硬件上
4. 以时间延时为代价的高吞吐量
5. 写入只支持单个写入，且修改只支持追加

### 概念

#### 数据块

hdfs文件划分为多个块，块作为独立存储单元。大小默认128M，目的为了最小化寻址开销，但也不能太大，map函数通常一次处理一个块的数据

#### 联邦HDFS

解决单个namenode内存限制，创建多个namenode，每个namenode负责文件系统命名空间的一部分，如/user,/share，

每个namenode维护一个命名空间卷，由命名空间的元数据和一个数据块池组成，命名空间卷相互独立，两两之间互不通信



## 3.1 概述

1. HDFS集群分为两大角色：NameNode、DataNode  (、Secondary Namenode)
2. NameNode负责管理整个文件系统的元数据
3. DataNode 负责管理用户的文件数据块
4. 文件会按照固定的大小（blocksize）切成若干块后分布式存储在若干台datanode上
5. 每一个文件块可以有多个副本，并存放在不同的datanode上
6. Datanode会定期向Namenode汇报自身所保存的文件block信息，而namenode则会负责保持文件的副本数量
7. HDFS的内部工作机制对客户端保持透明，客户端请求访问HDFS都是通过向namenode申请来进行

## 3.2 Namenode 和 Datanode

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

HDFS被设计成能够在一个大集群中跨机器可靠地存储超大文件。它将每个文件存储成一系列的数据块，除了最后一个，所有的数据块都是同样大小的。为了容错，文件的所有数据块都会有副本。每个文件的数据块大小和副本系数都是可配置的。副本系数可以在文件创建的时候指定，也可以在之后改变。==HDFS中的文件都是一次性写入的，并且严格要求在任何时候只能有一个写入者==。

Namenode全权管理数据块的复制，它周期性地从集群中的每个datanode接收心跳信号和块状态报告(Blockreport)。接收到心跳信号意味着该Datanode节点工作正常。块状态报告包含了一个该Datanode上所有数据块的列表。

> ###### HDFS的一致性分析
>
> 1. 为什么HDFS不支持多个writer同时写一个文件,即不支持并发写? 
>
>    多个reducer对同一文件执行写操作,即多个writer同时向HDFS的同一文件执行写操作, 这需要昂贵的同步机制不说, 最重要的是这种做法将各reducer的写操作顺序化, 不利于各reduce任务的并行。

### 3.4.1 副本存放

副本的存放是HDFS可靠性和性能的关键。HDFS采用一种称为机架感知(rack-aware)的策略来改进数据的可靠性、可用性和网络带宽的利用率。

通过一个[机架感知](http://hadoop.apache.org/docs/r1.0.4/cn/cluster_setup.html#Hadoop%E7%9A%84%E6%9C%BA%E6%9E%B6%E6%84%9F%E7%9F%A5)的过程，Namenode可以确定每个Datanode所属的机架id。将副本存放在不同的机架上，有效防止当整个机架失效时数据的丢失，但是，因为这种策略的一个写操作需要传输数据块到多个机架，这增加了写的代价。 

在大多数情况下，副本系数是3，HDFS的存放策略是:

1.  如果写请求方所在机器是其中一个datanode,则直接存放在本地,否则随机在集群中选择一个datanode.
2. 第二个副本存放于不同第一个副本的所在的机架.
3. 第三个副本存放于第二个副本所在的机架,但是属于不同的节点.
4. 其他副本均匀分布在剩下的机架中。如果replication factor大于3，则第4个以后的数据随机放在其他机架上，但保证每个机架副本数<(replicas - 1) / racks + 2

![这里写图片描述](https://img-blog.csdn.net/20160418112147560)

### 3.4.2 副本选择

为了降低整体的带宽消耗和读取延时，HDFS会尽量让读取程序读取离它最近的副本。

### 3.4.3 安全模式

Namenode启动后会进入一个称为安全模式的特殊状态。处于安全模式的Namenode是不会进行数据块的复制的。Namenode从所有的Datanode接收心跳信号和块状态报告。块状态报告包括了某个Datanode所有的数据块列表。每个数据块都有一个指定的最小副本数。当Namenode检测确认某个数据块的副本数目达到这个最小值，那么该数据块就会被认为是副本安全(safely replicated)的；在一定百分比（这个参数可配置）的数据块被Namenode检测确认是安全之后（加上一个额外的30秒等待时间），Namenode将退出安全模式状态。接下来它会确定还有哪些数据块的副本没有达到指定数目，并将这些数据块复制到其他Datanode上。

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

![img](https://img-blog.csdn.net/20150530212722149)

​      

1. 初始化FileSystem，客户端调用create()来创建文件
2. FileSystem用RPC调用元数据节点，在文件系统的命名空间中创建一个新的文件，元数据节点首先确定文件原来不存在，并且客户端有创建文件的权限，然后创建新文件。
3. FileSystem返回FSDataOutputStream，客户端用于写数据，客户端开始写入数据。FSDataOutputStream封装了DFSOutputStream，用于和datanode通信
4. DFSOutputStream将数据分成块，写入data queue。data queue由DataStreamer处理，并通知元数据节点分配数据节点，用来存储数据块(每块默认复制3块)。分配的数据节点放在一个pipeline里。DataStreamer将数据块写入pipeline中的第一个数据节点。第一个数据节点将数据块发送给第二个数据节点。第二个数据节点将数据发送给第三个数据节点。
5. DFSOutputStream为发出去的数据包保存了ack queue，收到pipeline中的==所有==数据节点确认信息才删除数据包。
6. 当客户端结束写入数据，则调用stream的close函数。此操作将所有的数据块写入pipeline中的数据节点，并等待ack queue返回成功。
7. 通知元数据节点写入完毕。

> 容错机制：
>
> 1. 多个client同时写入一个文件
>
>    写入时会在命名空间新建一个文件，但该文件还没有相应数据块。不需要考虑文件写入失败需要删除文件，因为是先创建文件再写入的，参考window写word
>
> 2. 写文件时datanode异常
>
>    如果数据节点在写入的过程中失败，关闭pipeline，将ack queue中的数据块放入data queue的开始，已经写入的错误数据节点中被元数据节点赋予新的标示，则错误节点重启后能够察觉其数据块是过时的，会被删除。失败的数据节点从pipeline中移除，另外的数据块则写入pipeline中的另外两个数据节点。元数据节点则被通知此数据块是复制块数不足，将来会再创建第三份备份。只要满足最小副本要求即可

## 3.10 HDFS读文件流程

客户端将要读取的文件路径发送给namenode，namenode获取文件的元信息（主要是block的存放位置信息）返回给客户端，客户端根据返回的信息找到相应datanode逐个获取文件的block并在客户端本地进行数据追加合并从而获得整个文件

![img](https://img-blog.csdn.net/20150522162524180)

1. 初始化FileSystem，然后客户端(client)用FileSystem的open()函数打开文件


2. FileSystem用RPC调用元数据节点，得到文件的数据块信息，对于每一个数据块，元数据节点返回保存数据块的数据节点的地址。
3. FileSystem返回FSDataInputStream给客户端，用来读取数据，客户端调用stream的read()函数开始读取数据。
4. DFSInputStream连接保存此文件第一个数据块的最近的数据节点，data从数据节点读到客户端(client)
5. 当此数据块读取完毕时，DFSInputStream关闭和此数据节点的连接，然后连接此文件下一个数据块的最近的数据节点。
6. 当客户端读取完毕数据的时候，调用FSDataInputStream的close函数。
7. 在读取数据的过程中，如果客户端在与数据节点通信出现错误，则尝试连接包含此数据块的下一个数据节点。
       8. 失败的数据节点将被记录，以后不再连接。【注意：这里的序号不是一一对应的关系】

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

1. SecondaryNameNode通知NameNode准备提交edits文件，此时主节点将新的写操作数据记录到一个新的文件edits.new中。 
2. SecondaryNameNode通过HTTP GET方式获取NameNode的fsimage与edits文件（在SecondaryNameNode的current同级目录下可见到 temp.check-point或者previous-checkpoint目录，这些目录中存储着从namenode拷贝来的镜像文件）。 
3. SecondaryNameNode开始合并获取的上述两个文件，产生一个新的fsimage文件fsimage.ckpt。 
4. SecondaryNameNode用HTTP POST方式发送fsimage.ckpt至NameNode。 
5. NameNode将fsimage.ckpt与edits.new文件分别重命名为fsimage与edits，然后更新fstime，整个checkpoint过程到此结束。 

![这里写图片描述](https://img-blog.csdn.net/20170503103823737?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveWFuZ2pqdWFu/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

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
   > ```xml
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

## 3.13 一致模型

HDFS 为了性能牺牲了一部分POSIX要求，文件读写的可见性有所不同

- 默认同flush()
- flush()不立即可见，数据超过一个块后，第一个块才可见，之后的块也一样
- hflush()立即可见，但仅保证所有数据传输到所有datanode的内存（pipeline）中
- hsync()同hflush()，但所有数据同步到磁盘中
- close()，同flush()


## 3.14 目录结构 

### namenode目录结构

``` java
├── current
│   ├── edits_0000000000000000001-0000000000000000428
│   ├── edits_0000000000000000428-0000000000000000429
│   ├── edits_inprogress_0000000000000000430
│   ├── fsimage_0000000000000000427
│   ├── fsimage_0000000000000000427.md5
│   ├── fsimage_0000000000000000429
│   ├── fsimage_0000000000000000429.md5
│   ├── seen_txid
│   └── VERSION
└── in_use.lock
```

- VERSION包含hdfs版本信息

  ``` properties
  #Thu May 10 14:24:22 CST 2018
  namespaceID=1435979192	# 文件系统命名空间唯一标识
  clusterID=CID-da2d62ad-2fd2-4a2c-b7f9-aa1a8e07915a	# hdfs集群唯一标识，对联邦hdfs很重要
  cTime=0	# 系统创建时间
  storageType=NAME_NODE	# 存储类型
  blockpoolID=BP-1294332222-192.168.199.163-1525600695333		# 数据块池唯一标识，包含namenode包含管理的所有文件
  layoutVersion=-60 	#描述hdfs持久化数据结构的版本
  ```

- in_use.lock：namenode使用该文件对存储目录加锁，文本内容是本机hadoop的pid

- edits_*：编辑日志，写操作会先记录到日志中。编辑日志修改时，相关元数据也会修改

- edits_inprogress_*：任何时刻只有一个文件处于打开状态，写操作时只有当每个事务完成后才更新该文件

- fsimage_*：元数据一个完整的永久检查点

- seen_txid：存放transactionId的文件，format之后是0，它代表的是namenode里面的edits_inprogress_*文件的尾数

### datanode目录结构

```
├── current
│   ├── BP-1294332222-192.168.199.163-1525600695333
│   │   ├── current
│   │   │   ├── finalized
│   │   │   │   └── subdir0
│   │   │   │       └── subdir0
│   │   │   │           ├── blk_1073741829
│   │   │   │           ├── blk_1073741829_1005.meta
│   │   │   │           ├── blk_1073741830
│   │   │   │           ├── blk_1073741830_1006.meta
│   │   │   ├── rbw
│   │   │   └── VERSION
│   │   ├── scanner.cursor
│   │   └── tmp
│   └── VERSION
└── in_use.lock


```

- hdfs数据块存储在以blk_为前缀的文件中，包含该块的原始字节数，并关联一个.meta的元数据文件，包括头部和该块各区段一系列的校验和。
- BP-*：每个块属于一个数据块池，每个数据块池都有自己的目录，目录是namenode的VERSION中的数据块池id
- finalized保存所有的块，当块数量过多，会创建子目录，默认64，方便管理
- in_use.lock同namenode



