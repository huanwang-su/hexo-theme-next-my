---
title: spark运行模式
date: 2018/5/12 15:30:25
category:
- 大数据
- spark
tag:
- spark
comments: true  
---

# spark运行模式

Spark 有很多种模式，最简单就是单机本地模式，还有单机伪分布式模式，复杂的则运行在集群中，目前能很好的运行在 Yarn和 Mesos 中

 - local(本地模式)：常用于本地开发测试，本地还分为local单线程和local-cluster多线程;
 - standalone(集群模式)：典型的Mater/slave模式，不过也能看出Master是有单点故障的；Spark支持ZooKeeper来实现 HA
 - on yarn(集群模式)： 运行在 yarn 资源管理器框架之上，由 yarn 负责资源管理，Spark 负责任务调度和计算
 - on mesos(集群模式)： 运行在 mesos 资源管理器框架之上，由 mesos 负责资源管理，Spark 负责任务调度和计算
 - on cloud(集群模式)：比如 AWS 的 EC2，使用这个模式能很方便的访问 Amazon的 S3;Spark 支持多种分布式存储系统：HDFS 和 S3

## local

### 本地单机模式

本地单机模式下，所有的Spark进程均运行于同一个JVM中，并行处理则通过多线程来实现。在默认情况下，单机模式启动与本地系统的CPU核心数目相同的线程。如果要设置并行的级别，则以local[N]的格式来指定一个master变量，N表示要使用的线程数目。

运行该模式非常简单，只需要把Spark的安装包解压后，改一些常用的配置即可使用，而不用启动Spark的Master、Worker守护进程( 只有集群的Standalone方式时，才需要这两个角色)，也不用启动Hadoop的各服务（除非你要用到HDFS）

``` shell
spark-submit --class JavaWordCount --master local[10] JavaWordCount.jar file:///tmp/test.txt 
```

SparkSubmit进程既是客户提交任务的Client进程、又是Spark的driver程序、还充当着Spark执行Task的Executor角色。

因为driver程序在应用程序结束后就会终止，要想看webui可以Thread.sleep()

### 本地伪集群运行模式

这种运行模式，和Local[N]很像，不同的是，它会在单机启动多个进程来模拟集群下的分布式场景

用法是：提交应用程序时使用local-cluster[x,y,z]参数：x代表要生成的executor数，y和z分别代表每个executor所拥有的core和memory数。

``` shell
spark-submit --master local-cluster[2, 3, 1024]
```

SparkSubmit依然充当全能角色，又是Client进程，又是driver程序

## 集群模式介绍

为了运行在集群上，SparkContext 可以连接至几种类型的 Cluster Manager（既可以用 Spark 自己的 Standlone Cluster Manager，或者 Mesos，也可以使用 YARN），它们会分配应用的资源。一旦连接上，Spark 获得集群中节点上的 Executor，这些进程可以运行计算并且为您的应用存储数据。接下来，它将发送您的应用代码（通过 JAR 或者 Python 文件定义传递给 SparkContext）至 Executor。最终，SparkContext 将发送 Task 到 Executor 以运行。

![Spark cluster components](http://spark.apachecn.org/docs/cn/2.2.0/img/cluster-overview.png) 

1. 每个应用获取到它自己的 Executor 进程，它们会保持在整个应用的生命周期中并且在多个线程中运行 Task（任务）。这样做的优点是把应用互相隔离，在调度方面（每个 driver 调度它自己的 task）和 Executor 方面（来自不同应用的 task 运行在不同的 JVM 中）。然而，这也意味着若是不把数据写到外部的存储系统中的话，数据就不能够被不同的 Spark 应用（SparkContext 的实例）之间共享。
2. Spark 是不知道底层的 Cluster Manager 到底是什么类型的。只要它能够获得 Executor 进程，并且它们可以和彼此之间通信，那么即使是在一个也支持其它应用的 Cluster Manager（例如，Mesos / YARN）上来运行它也是相对简单的。
3. Driver 程序必须在自己的生命周期内监听和接受来自它的 Executor 的连接请求。同样的，driver 程序必须可以从 worker 节点上网络寻址。
4. 因为 driver 调度了集群上的 task（任务），更好的方式应该是在相同的局域网中靠近 worker 的节点上运行。

### 提交应用程序

使用 spark-submit 脚本可以提交应用至任何类型的集群。

### 监控

每个 driver 都有一个 Web UI，通常在端口 4040 上，可以显示有关正在运行的 task，executor，和存储使用情况的信息。 只需在 Web 浏览器中的`http://<driver-node>:4040` 中访问此 UI。

### Job 调度

Spark 即可以在应用间（Cluster Manager 级别），也可以在应用内（如果多个计算发生在相同的 SparkContext 上时）控制资源分配。 

### 术语

| Term（术语）    | Meaning（含义）                                              |
| --------------- | ------------------------------------------------------------ |
| Application     | 用户构建在 Spark 上的程序。由集群上的一个 driver 程序和多个 executor 组成。 |
| Application jar | 一个包含用户 Spark 应用的 Jar。有时候用户会想要去创建一个包含他们应用以及它的依赖的 “uber jar”。用户的 Jar 应该没有包括 Hadoop 或者 Spark 库，然而，它们将会在运行时被添加。 |
| Driver program  | 该进程运行应用的 main() 方法并且创建了 SparkContext。        |
| Cluster manager | 一个外部的用于获取集群上资源的服务。（例如，Standlone Manager，Mesos，YARN） |
| Deploy mode     | 根据 driver 程序运行的地方区别。在 “Cluster” 模式中，框架在群集内部启动 driver。在 “Client” 模式中，submitter（提交者）在 Custer 外部启动 driver。 |
| Worker node     | 任何在集群中可以运行应用代码的节点。                         |
| Executor        | 一个为了在 worker 节点上的应用而启动的进程，它运行 task 并且将数据保持在内存中或者硬盘存储。每个应用有它自己的 Executor。 |
| Task            | 一个将要被发送到 Executor 中的工作单元。                     |
| Job             | 一个由多个任务组成的并行计算，并且能从 Spark action 中获取响应（例如 `save`, `collect`）; 您将在 driver 的日志中看到这个术语。 |
| Stage           | 每个 Job 被拆分成更小的被称作 stage（阶段） 的 task（任务） 组，stage 彼此之间是相互依赖的（与 MapReduce 中的 map 和 reduce stage 相似）。您将在 driver 的日志中看到这个术语。 |

## standalone

在执行应用程序前，先启动Spark的Master和Worker守护进程。

``` shell
spark-submit --master spark://wl1:7077

或者 spark-submit --master spark://wl1:7077 --deploy-mode client
```

会在所有有Worker进程的节点上启动Executor来执行应用程序，此时产生的JVM进程如下：（非master节点，除了没有Master、SparkSubmit，其他进程都一样）

![img](http://upload-images.jianshu.io/upload_images/5432088-bba9dab295abbce0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700) 

> Master进程做为cluster manager，用来对应用程序申请的资源进行管理；
>
> SparkSubmit 做为Client端和运行driver程序；
>
> CoarseGrainedExecutorBackend 用来并发执行应用程序；
>
> 注意，Worker进程生成几个Executor，每个Executor使用几个core，这些都可以在spark-env.sh里面配置

![img](http://upload-images.jianshu.io/upload_images/5432088-a9094822ca8cd8db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700) 

### 过程

1. SparkContext连接到Master，向Master注册并申请资源（CPU Core 和Memory）；
2. Master根据SparkContext的资源申请要求和Worker心跳周期内报告的信息决定在哪个Worker上分配资源，然后在该Worker上获取资源，然后启动StandaloneExecutorBackend；
3. StandaloneExecutorBackend向SparkContext注册；
4. SparkContext将Applicaiton代码发送给StandaloneExecutorBackend；并且SparkContext解析Applicaiton代码，构建DAG图，并提交给DAG Scheduler分解成Stage（当碰到Action操作时，就会催生Job；每个Job中含有1个或多个Stage，Stage一般在获取外部数据和shuffle之前产生），然后以Stage（或者称为TaskSet）提交给Task Scheduler，Task Scheduler负责将Task分配到相应的Worker，最后提交给StandaloneExecutorBackend执行；
5. StandaloneExecutorBackend会建立Executor线程池，开始执行Task，并向SparkContext报告，直至Task完成。
6. 所有Task完成后，SparkContext向Master注销，释放资源。

![clip_image016](http://www.th7.cn/d/file/p/2017/08/14/99909df840cd7e773dff2f483fe3594b.jpg)

## YARN

Spark on YARN模式根据Driver在集群中的位置分为两种模式：一种是YARN-Client模式，另一种是YARN-Cluster（或称为YARN-Standalone模式）。

由于已经有了资源管理器，所以不需要启动Spark的Master、Worker守护进程。

``` shell
$ ./bin/spark-submit --class org.apache.spark.examples.SparkPi \
    --master yarn \ # --deploy-mode client
    --deploy-mode cluster \
    --driver-memory 4g \
    --executor-memory 2g \
    --executor-cores 1 \
    --queue thequeue \
    lib/spark-examples*.jar \
    10
```

### YARN框架流程 

任何框架与YARN的结合，都必须遵循YARN的开发模式。

Yarn框架的基本运行流程图为：

![clip_image018](http://www.th7.cn/d/file/p/2017/08/14/ec9386134f075aef2adcdb49b97c4b65.jpg) 

其中，ResourceManager负责将集群的资源分配给各个应用使用，而资源分配和调度的基本单位是Container，其中封装了机器资源，如内存、CPU、磁盘和网络等，每个任务会被分配一个Container，该任务只能在该Container中执行，并使用该Container封装的资源。NodeManager是一个个的计算节点，主要负责启动Application所需的Container，监控资源（内存、CPU、磁盘和网络等）的使用情况并将之汇报给ResourceManager。ResourceManager与NodeManagers共同组成整个数据计算框架，ApplicationMaster与具体的Application相关，主要负责同ResourceManager协商以获取合适的Container，并跟踪这些Container的状态和监控其进度。

### YARN-Client 

Yarn-Client模式中，Driver在客户端本地运行

YARN-client的工作流程分为以下几个步骤：

![clip_image020](http://www.th7.cn/d/file/p/2017/08/14/a0ad1f86ae4ab1059865377c333d13d0.jpg)

1. Spark Yarn Client向YARN的ResourceManager申请启动Application Master。同时在SparkContent初始化中将创建DAGScheduler和TASKScheduler等，由于我们选择的是Yarn-Client模式，程序会选择YarnClientClusterScheduler和YarnClientSchedulerBackend；
2. ResourceManager收到请求后，在集群中选择一个NodeManager，为该应用程序分配第一个Container，要求它在这个Container中启动应用程序的ApplicationMaster，与YARN-Cluster区别的是在该ApplicationMaster不运行SparkContext，只与SparkContext进行联系进行资源的分派；
3. Client中的SparkContext初始化完毕后，与ApplicationMaster建立通讯，向ResourceManager注册，根据任务信息向ResourceManager申请资源（Container）；
4. 一旦ApplicationMaster申请到资源（也就是Container）后，便与对应的NodeManager通信，要求它在获得的Container中启动启动CoarseGrainedExecutorBackend，CoarseGrainedExecutorBackend启动后会向Client中的SparkContext注册并申请Task；
5. Client中的SparkContext分配Task给CoarseGrainedExecutorBackend执行，CoarseGrainedExecutorBackend运行Task并向Driver汇报运行的状态和进度，以让Client随时掌握各个任务的运行状态，从而可以在任务失败时重新启动任务；
6. 应用程序运行完成后，Client的SparkContext向ResourceManager申请注销并关闭自己。

### YARN-Cluster 

在YARN-Cluster模式中，当用户向YARN中提交一个应用程序后，YARN将分两个阶段运行该应用程序：第一个阶段是把Spark的Driver作为一个ApplicationMaster在YARN集群中先启动；第二个阶段是由ApplicationMaster创建应用程序，然后为它向ResourceManager申请资源，并启动Executor来运行Task，同时监控它的整个运行过程，直到运行完成。

YARN-cluster的工作流程分为以下几个步骤：

![clip_image022](http://www.th7.cn/d/file/p/2017/08/14/70b60a727d521d8cd502c6dc7f8eb9e0.jpg)

1. Spark Yarn Client向YARN中提交应用程序，包括ApplicationMaster程序、启动ApplicationMaster的命令、需要在Executor中运行的程序等；
2. ResourceManager收到请求后，在集群中选择一个NodeManager，为该应用程序分配第一个Container，要求它在这个Container中启动应用程序的ApplicationMaster，其中ApplicationMaster进行SparkContext等的初始化；
3. ApplicationMaster向ResourceManager注册，这样用户可以直接通过ResourceManage查看应用程序的运行状态，然后它将采用轮询的方式通过RPC协议为各个任务申请资源，并监控它们的运行状态直到运行结束；
4. 一旦ApplicationMaster申请到资源（也就是Container）后，便与对应的NodeManager通信，要求它在获得的Container中启动启动CoarseGrainedExecutorBackend，CoarseGrainedExecutorBackend启动后会向ApplicationMaster中的SparkContext注册并申请Task。这一点和Standalone模式一样，只不过SparkContext在Spark Application中初始化时，使用CoarseGrainedSchedulerBackend配合YarnClusterScheduler进行任务的调度，其中YarnClusterScheduler只是对TaskSchedulerImpl的一个简单包装，增加了对Executor的等待逻辑等；
5. ApplicationMaster中的SparkContext分配Task给CoarseGrainedExecutorBackend执行，CoarseGrainedExecutorBackend运行Task并向ApplicationMaster汇报运行的状态和进度，以让ApplicationMaster随时掌握各个任务的运行状态，从而可以在任务失败时重新启动任务；
6. 应用程序运行完成后，ApplicationMaster向ResourceManager申请注销并关闭自己。

### YARN-Client 与 YARN-Cluster 区别 

理解YARN-Client和YARN-Cluster深层次的区别之前先清楚一个概念：Application Master。==在YARN中，每个Application实例都有一个ApplicationMaster进程，它是Application启动的第一个容器==。它负责和ResourceManager打交道并请求资源，获取资源之后告诉NodeManager为其启动Container。从深层次的含义讲YARN-Cluster和YARN-Client模式的区别其实就是ApplicationMaster进程的区别。

- YARN-Cluster模式下，Driver运行在AM(Application Master)中，它负责向YARN申请资源，并监督作业的运行状况。当用户提交了作业之后，就可以关掉Client，作业会继续在YARN上运行，因而YARN-Cluster模式不适合运行交互类型的作业；
- YARN-Client模式下，Application Master仅仅向YARN请求Executor，Client会和请求的Container通信来调度他们工作，也就是说Client不能离开。

## Mesos

Apache Mesos是一个通用的集群管理器

``` shell
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master mesos://207.184.161.138:7077 \
  --deploy-mode cluster \
  --supervise \
  --executor-memory 20G \
  --total-executor-cores 100 \
  http://path/to/examples.jar \
  1000
```

