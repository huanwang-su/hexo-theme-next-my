---
title: spark应用执行机制
date: 2018/5/12 15:30:25
category:
- 大数据
- spark
tag:
- spark
comments: true  
---

# spark应用执行机制

## 执行机制总览

action算子触发job提交，提交到spark的job生成RDD DAG，经过DAGScheduler转化为stage DAG，每个stage中产生相应的task集合，taskscheduler讲任务分发到executor执行。每个任务对应相应的一个数据块，使用用户定义的函数处理数据。

![ img](https://img-blog.csdn.net/20160912175158865?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center) 

spark为了系统的内存不会快速用完，使用延迟执行，只有操作累积到action，算子才会触发整个操作序列的执行，中间结果不会单独

重新分配内存，而是在同一个数据块上进行流水线操作。

对RDD的块管理通过BlockManager完成，BlockManager将数据抽象为数据块，在内存或者磁盘进行存储，如果数据不在本节点，则还可以通过远端节点复制到本机进行计算。

![img](https://img-blog.csdn.net/20170307161800385?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvSG9sZG9uV2l0aFlvdXJHb2Fs/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center) 

在计算节点的执行器Exector中会创建线程池，这个执行器将需要执行的任务通过线程池进行并发执行。

## Spark应用概念 

- Application：Spark的应用程序，用户提交后，Spark为App分配资源，将程序转换并执行，其中Application包含一个Drive Program和若干个Executor。

- Driver Progream： 运行Application的main()函数，并创建SparkContext。

- SparkContext：Spark程序的入口，负责调度各个运算资源，协调各个worker Node上的Executor。

- RDD Graph：RDD是Spark的核心结构，可以通过一系列算子进行操作（主要有Transformation和Action操作）。当RDD遇到Action算子时，将之前所有的算子形成一个有向无形环（DAG），再在Spark中转化为Job，提交到集群执行。一个App中可以包含多个Job。

- Job：一个RDD Graph触发的作业，往往由Spark Action触发算子，在SparkContext中通过runJob方法想Spark提交Job。

- Stage：每个Job会根据RDD的依赖关系被切分为很多歌Stage，每个Stage中会包含一组相同的Task，这一组Task也叫TaskSet

- Task：一个分区对应一个Task，Task执行RDD中对应的Stage包含的算子，Task被封装好后放入Exector的线程池中执行

- DAGScheduler：根据Job构建基于Stage的DAG，并提交Stage给TaskSheduler。

- TaskScheduler：将TaskSet提交给worker Node集群并返回结果。

  ![img](https://img-blog.csdn.net/20170307143921093) 

  ## 应用提交与执行方式

  以standalone为例

  ### driver进程运行在客户端，对应用进行管理监控

  1. 启动集群，Master和Worker，Worker向Master注册
  2. 客户端启动后直接运行用户程序，启动Driver相关的工作：DAGScheduler和BlockManagerMaster等。客户端的Driver向Master注册。
  3. Master让Worker启动Exeuctor。Worker创建一个ExecutorRunner线程，ExecutorRunner会启动ExecutorBackend进程。
  4. ExecutorBackend启动后会向Driver的SchedulerBackend注册，Driver就可以找到计算资源。Driver的DAGScheduler解析作业并生成相应的Stage，每个Stage包含的Task通过TaskScheduler分配给Executor执行。所有stage都完成后作业结束 

  ![img](https://img-blog.csdn.net/20160912183017512?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center) 

  ### Driver在worker运行

  1. 启动集群，Master和Worker，Worker向Master注册
  2. 客户端提交作业给Master，Master让一个Worker启动Driver，即SchedulerBackend。Worker创建一个DriverRunner线程，DriverRunner启动SchedulerBackend进程。
  3. 其余同上

  ![img](https://img-blog.csdn.net/20160912183515650?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center) 

  ​

  ## Spark运行基本流程

  1. 构建Spark Application的运行环境（启动SparkContext），SparkContext向资源管理器（可以是Standalone、Mesos或YARN）注册并申请运行Executor资源；
  2. 资源管理器分配Executor资源并启动StandaloneExecutorBackend，Executor运行情况将随着心跳发送到资源管理器上；
  3. SparkContext构建成DAG图，将DAG图分解成Stage，并把Taskset发送给Task Scheduler。Executor向SparkContext申请Task，Task Scheduler将Task发放给Executor运行同时SparkContext将应用程序代码发放给Executor。
  4. Task在Executor上运行，运行完毕释放所有资源。

  ![clip_image004](http://www.th7.cn/d/file/p/2017/08/14/d9417f89faf74ca75473db885e02b993.jpg) 

### Spark运行架构特点：

- 每个Application获取专属的executor进程，该进程在Application期间一直驻留，并以多线程方式运行tasks。这种Application隔离机制有其优势的，无论是从调度角度看（每个Driver调度它自己的任务），还是从运行角度看（来自不同Application的Task运行在不同的JVM中）。当然，这也意味着Spark Application不能跨应用程序共享数据，除非将数据写入到外部存储系统。
- Spark与资源管理器无关，只要能够获取executor进程，并能保持相互通信就可以了。
- 提交SparkContext的Client应该靠近Worker节点（运行Executor的节点)，最好是在同一个Rack里，因为Spark Application运行过程中SparkContext和Executor之间有大量的信息交换；如果想在远程集群中运行，最好使用RPC将SparkContext提交给集群，不要远离Worker运行SparkContext。
- Task采用了数据本地性和推测执行的优化机制。

### DAGScheduler 

DAGScheduler把一个Spark作业转换成Stage的DAG（Directed Acyclic Graph有向无环图），根据RDD和Stage之间的关系找出开销最小的调度方法，然后把Stage以TaskSet的形式提交给TaskScheduler，下图展示了DAGScheduler的作用：

![clip_image006](http://www.th7.cn/d/file/p/2017/08/14/c9ccefd4c3370be5ddb275abb9cd4781.jpg) 

### TaskScheduler 

DAGScheduler决定了运行Task的理想位置，并把这些信息传递给下层的TaskScheduler。此外，DAGScheduler还处理由于Shuffle数据丢失导致的失败，这有可能需要重新提交运行之前的Stage（非Shuffle数据丢失导致的Task失败由TaskScheduler处理）

TaskScheduler维护所有TaskSet，当Executor向Driver发送心跳时，TaskScheduler会根据其资源剩余情况分配相应的Task。另外TaskScheduler还维护着所有Task的运行状态，重试失败的Task。下图展示了TaskScheduler的作用：

![clip_image008](http://www.th7.cn/d/file/p/2017/08/14/09df73e1c436b04124a084bd6312402d.jpg) 

在不同运行模式中任务调度器具体为：

- Spark on Standalone模式为TaskScheduler；


- YARN-Client模式为YarnClientClusterScheduler
- YARN-Cluster模式为YarnClusterScheduler

### RDD运行原理 

高层次来看，主要分为三步：

1. 创建 RDD 对象
2. DAGScheduler模块介入运算，计算RDD之间的依赖关系。RDD之间的依赖关系就形成了DAG
3. 每一个JOB被分为多个Stage，划分Stage的一个主要依据是当前计算因子的输入是否是确定的，如果是则将其分在同一个Stage，避免多个Stage之间的消息传递开销。

![clip_image010](http://www.th7.cn/d/file/p/2017/08/14/ff86663e6cf60eb41be88c7d003c2c5d.jpg) 



