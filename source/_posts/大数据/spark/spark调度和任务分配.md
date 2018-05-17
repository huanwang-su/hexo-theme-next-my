---
title: spark调度和任务分配
date: 2018/5/12 15:30:25
category:
- 大数据
- spark
tag:
- spark
comments: true  
---

# spark执行流程

### 基本概念

- Application：表示你的应用程序
- Driver：表示main()函数，创建SparkContext。由SparkContext负责与ClusterManager通信，进行资源的申请，任务的分配和监控等。程序执行完毕后关闭SparkContext
- Executor：某个Application运行在Worker节点上的一个进程，该进程负责运行某些task，并且负责将数据存在内存或者磁盘上。在Spark on Yarn模式下，其进程名称为CoarseGrainedExecutor Backend，一个CoarseGrainedExecutor Backend进程有且仅有一个executor对象，它负责将Task包装成taskRunner，并从线程池中抽取出一个空闲线程运行Task，这样，每个CoarseGrainedExecutorBackend能并行运行Task的数据就取决于分配给它的CPU的个数。
- Worker：集群中可以运行Application代码的节点。在Standalone模式中指的是通过slave文件配置的worker节点，在Spark on Yarn模式中指的就是NodeManager节点。
- Task：在Executor进程中执行任务的工作单元，多个Task组成一个Stage
- Job：包含多个Task组成的并行计算，是由Action行为触发的
- Stage：每个Job会被拆分很多组Task，作为一个TaskSet，其名称为Stage
- DAGScheduler：根据Job构建基于Stage的DAG，并提交Stage给TaskScheduler，其划分Stage的依据是RDD之间的依赖关系
- TaskScheduler：将TaskSet提交给Worker（集群）运行，每个Executor运行什么Task就是在此处分配的。


### Spark运行基本流程

1. 构建Spark Application的运行环境（启动SparkContext），SparkContext向资源管理器（可以是Standalone、Mesos或YARN）注册并申请运行Executor资源；
2. 资源管理器分配Executor资源并启动StandaloneExecutorBackend，Executor运行情况将随着心跳发送到资源管理器上；
3. SparkContext构建成DAG图，将DAG图分解成Stage，并把Taskset发送给Task Scheduler。Executor向SparkContext申请Task，Task Scheduler将Task发放给Executor运行同时SparkContext将应用程序代码发放给Executor。
4. Task在Executor上运行，运行完毕释放所有资源。

![clip_image004](http://www.th7.cn/d/file/p/2017/08/14/d9417f89faf74ca75473db885e02b993.jpg) 


### 调度

调度可以分为4个级别，Application调度、Job调度、Stage的调度、Task的调度与分发

## 应用调度

一个executor在一个时间段内只能分配给一个应用使用。

- Standalong，采用FIFO模式，每个应用会独占所有可用节点的资源
  - spark.cores.max：调整app可以在整个集群中申请的CPU core数量
  - spark.deploy.defaultCores：默认的CPU core数量
  - spark.executor.memory：限制每个Executor可用的内存
- Mesos，用户可以通过参数设置所提交应用占用的资源
  - spark.mesos.coarse=true设置静态配置资源的策略
  - 使用mesos://URL且不配置spark.mesos.coarse=true（每个app会有独立固定的内存分配，空闲时其他机器可以使用其资源）
- Yarn，可以指定为应用分配多少Executor，每个Executer的cpu和内存大小等，每个应用不会占用过多资源，提高yarn的吞吐量
  - 通过–num-executors控制分配多少个Executor给app
  - –executor-memory和–executor-cores分别控制Executor的内存和CPU core

## Job调度

Spark应用程序内部，用户通过不同线程提交的Job可以并行运行，这里所说的Job就是Spark Action（如count、collect等）算子触发的整个RDD DAG为一个Job，在实现上，算子中的本质是调用SparkContext中的runJob提交了Job。

### FIFO模式

- 第一个Job分配其所需的所有资源
- 第二个Job如果还有剩余资源的话就分配，否则等待

### FAIR

- 使用轮询的方式调度Job

### 配置调度池

默认新任务在 default pool中调度，可以在sc中设置：

```scala
sc.setLocalProperty("spark.scheduler.pool","pool6")
```

设置该参数线程每次提交任务都放在这个池中调度，可以设置`sc.setLocalProperty("spark.scheduler.pool","pool6")`终止这个调度池的使用

默认每个调度池拥有相同优先级来共享整个集群的资源，可以通过一下配置

``` xml
<?xml version="1.0"?>
<allocations>
  <pool name="production">
    <schedulingMode>FAIR</schedulingMode>
    <weight>1</weight>
    <minShare>2</minShare>
  </pool>
  <pool name="test">
    <schedulingMode>FIFO</schedulingMode>
    <weight>2</weight>
    <minShare>3</minShare>
  </pool>
</allocations>
```

- schedulingMode：FAIR或者FIFO。


- weight： 权重，默认是1，设置为2的话，就会比其他调度池获得2x多的资源，如果设置为-1000，该调度池一有任务就会马上运行。


- minShare： 最小共享核心数，默认是0，在权重相同的情况下，minShare大的，可以获得更多的资源。

我们可以通过spark.scheduler.allocation.file参数来设置这个文件的位置。

```
System.setProperty("spark.scheduler.allocation.file", "/path/to/file")
```

## Stage调度

### stage的生成

由DAGScheduler完成。RDD的有向无环图DAG切分出了Stage的有向无环图DAG。Stage的DAG通过==最后执行的Stage为根==进行广度优先遍历，遍历到最开始执行的Stage执行，如果提交的Stage仍有未完成的父母Stage，则Stage需要等待其父Stage执行完才能执行。同时DAGScheduler中还维持了几个重要的Key-Value集合结构，用来记录Stage的状态，这样能够避免过早执行和重复提交Stage。waitingStages中记录仍有未执行的父母Stage，防止过早执行。runningStages中保存正在执行的Stage，防止重复执行。failedStages中保存执行失败的Stage，需要重新执行，这里的设计是出于容错的考虑。

#### 依赖关系

RDD的窄依赖是指父RDD的所有输出都会被指向的一个子RDD，即输出多对一；宽依赖是指父RDD的输出会指向不同的子RDD，即输出多对多。 
**调度器会计算RDD之间的依赖关系，将拥有持续窄依赖的RDD归并到同一个Stage中，而宽依赖则作为划分不同Stage的判断标准。** 

导致窄依赖的Transformation操作：map、flatMap、filter、sample；导致宽依赖的Transformation操作：sortByKey、reduceByKey、groupByKey、cogroupByKey、join、cartensian。 

#### Stage分为两种： 

##### ShuffleMapStage

in which case its tasks’ results are input for another stage 
其实就是,非最终stage, 后面还有其他的stage, 所以它的输出一定是需要shuffle并作为后续的输入。

这种Stage是以Shuffle为输出边界，其输入边界可以是从外部获取数据，也可以是另一个ShuffleMapStage的输出 其输出可以是另一个Stage的开始。 ShuffleMapStage的最后Task就是ShuffleMapTask。 

在一个Job里可能有该类型的Stage，也可以能没有该类型Stage。

##### ResultStage

in which case its tasks directly compute the action that initiated a job (e.g. count(), save(), etc) 
最终的stage, 没有输出, 而是直接产生结果或存储。

这种Stage是直接输出结果，其输入边界可以是从外部获取数据，也可以是另一个ShuffleMapStage的输出。 ResultStage的最后Task就是ResultTask，在一个Job里必定有该类型Stage。 一个Job含有一个或多个Stage，但至少含有一个ResultStage。

### Stage的划分

RDD转换本身存在ShuffleDependency，像ShuffleRDD、CoGroupdRDD、SubtractedRDD会返回ShuffleDependency。 如果RDD中存在ShuffleDependency，就会创建一个新的Stage。 
Stage划分完毕就明确了以下内容：

> 1. 产生的Stage需要从多少个Partition中读取数据
> 2. 产生的Stage会生成多少Partition
> 3. 产生的Stage是否属于ShuffleMap类型

确认Partition以决定需要产生多少不同的Task，ShuffleMap类型判断来决定生成的Task类型。Spark中有两种Task，分别是ShuffleMapTask和ResultTask。

### Stage类

stage的RDD参数只有一个RDD, final RDD, 而不是一系列的RDD。 
因为在==一个stage中的所有RDD都是map, partition不会有任何改变, 只是在data依次执行不同的map function==所以对于TaskScheduler而言, 一个RDD的状况就可以代表这个stage。

> Stage参数说明： 
> val id: Int //Stage的序号数值越大，优先级越高 
> val rdd: RDD[_], //归属于本Stage的最后一个rdd 
> val numTasks: Int, //创建的Task数目，等于父RDD的输出Partition数目 
> val shuffleDep: Option[ShuffleDependency[*,* , _]], //是否存在SuffleDependency，宽依赖 
> val parents: List[Stage], //父Stage列表 
> val jobId: Int //作业ID

### 处理Job，分割Job为Stage，封装Stage成TaskSet，最终提交给TaskScheduler的调用链

`dagScheduler.handleJobSubmitted`-->`dagScheduler.submitStage`-->`dagScheduler.submitMissingTasks`-->`taskScheduler.submitTasks`

#### handleJobSubmitted函数

函数handleJobSubmitted和submitStage主要负责依赖性分析，对其处理逻辑做进一步的分析。
handleJobSubmitted最主要的工作是生成Stage，并根据finalStage来产生ActiveJob。

==在创建一个Stage之前，需要知道该Stage需要从多少个Partition读入数据==，这个数值直接影响要创建多少个Task。也就是说，创建Stage时，已经清楚该Stage需要从多少不同的Partition读入数据，并写出到多少个不同的Partition中，输入和输出的个数均已明确。创建一个Stage的时候，通过不停的遍历它之前的rdd，如果碰到有依赖是ShuffleDependency类型的，就通过getShuffleMapStage方法计算出来它的Stage来。

##### ActiveJob类

用户所提交的job在得到DAGScheduler的调度后，会被包装成ActiveJob，同时会启动JobWaiter阻塞监听job的完成状况。

同时依据job中RDD的dependency和dependency属性(NarrowDependency，ShufflerDependecy)，DAGScheduler会根据依赖关系的先后产生出不同的stage DAG(result stage, shuffle map stage)。

在每一个stage内部，根据stage产生出相应的task，包括ResultTask或是ShuffleMapTask，这些task会根据RDD中partition的数量和分布，产生出一组相应的task，并将其包装为TaskSet提交到TaskScheduler上去。

``` scala
private[spark] class ActiveJob(
    val jobId: Int,
    val finalStage: Stage,
    val func: (TaskContext, Iterator[_]) => _,
    val partitions: Array[Int],
    val callSite: CallSite,
    val listener: JobListener,
    val properties: Properties) {

  val numPartitions = partitions.length
  val finished = Array.fill[Boolean](numPartitions)(false)
  var numFinished = 0
}
```

#### submitStage

通过submitStage方法提交finalStage，方法会递归地将finalStage依赖的父stage先提交，最后提交finalStage，具体代码如下：

```scala
/** Submits stage, but first recursively submits any missing parents. */
  private def submitStage(stage: Stage) {
    val jobId = activeJobForStage(stage)
    if (jobId.isDefined) {
      logDebug("submitStage(" + stage + ")")
      if (!waitingStages(stage) && !runningStages(stage) && !failedStages(stage)) {
        //获取依赖的未提交的父stage
        val missing = getMissingParentStages(stage).sortBy(_.id)
        logDebug("missing: " + missing)
        if (missing.isEmpty) {
          logInfo("Submitting " + stage + " (" + stage.rdd + "), which has no missing parents")
          //如果父Stage都提交完成，则提交Stage
          submitMissingTasks(stage, jobId.get)
        } else {
          //如果有未提交的父Stage，则递归提交
          for (parent <- missing) {
            submitStage(parent)
          }
          waitingStages += stage
        }
      }
    } else {
      abortStage(stage, "No active job for stage " + stage.id, None)
    }
  }
```

submitStage处理流程：

- 如果所有的依赖已经完成，则提交自身所处的Stage
- 最后会在submitMissingTasks函数中将stage封装成TaskSet通过taskScheduler.submitTasks函数提交给TaskScheduler处理。

#### dagScheduler.submitMissingTasks

无论是哪种stage，都是对于每个stage中的每个partitions创建task，并最终封装成TaskSet，将该stage提交给taskscheduler。

#### taskScheduler.submitTasks

见下文

## Task调度

stage划分完之后会以TaskSet的形式提交给我们的TaskScheduler。

![img](https://img-blog.csdn.net/20170413155623145?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcmFpbnR1bmdsaQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center) 



### TaskSetManager

每个Stage对应的一个TaskSetManager通过Stage回溯到最源头缺失的Stage提交到==调度池pool==中，在调度池中，这些TaskSetMananger又会根据Job ID排序，先提交的Job的TaskSetManager优先调度，然后一个Job内的TaskSetManager ID小的先调度，并且==如果有未执行完的父母Stage的TaskSetManager，则是不会提交到调度池中。==

整体的Task分发由TaskSchedulerImpl来实现，但是Task的调度（本质上是Task在哪个分区执行）逻辑由TaskSetManager完成。这个类监控整个任务的生命周期，当任务失败时（如执行时间超过一定的阈值），重新调度，也会通过delay scheduling进行基于位置感知（locality-aware）的任务调度。

### TaskSet的生成

ShuffleMapStage 转化成 ShuffleMapTask

ResultStage 转化成为 ResultTask

TaskSet类定义如下：

```scala
/**
 * A set of tasks submitted together to the low-level TaskScheduler, usually representing
 * missing partitions of a particular stage.
 */
private[spark] class TaskSet(
    val tasks: Array[Task[_]],
    val stageId: Int,
    val stageAttemptId: Int,
    val priority: Int,
    val properties: Properties) {
    val id: String = stageId + "." + stageAttemptId

  override def toString: String = "TaskSet " + id
}
```

生成方法是上面stage调度中的`submitMissingTasks(stage: Stage, jobId: Int) `，会设置每个任务累加变量，广播变量，任务序列化的二进制文件，任务执行位置等信息设置

```scala
/** Called when stage's parents are available and we can now do its task. */
  private def submitMissingTasks(stage: Stage, jobId: Int) {
	// ...
    // First figure out the indexes of partition ids to compute.
    val (allPartitions: Seq[Int], partitionsToCompute: Seq[Int]) = {
      stage match {
        //在DAG Stage依赖关系中，除之后的Stage 外，全部为ShuffleMapStage 
        //allPartitions为所有partion的ID
        //filteredPartitions为不在缓存中的partion ID
        case stage: ShuffleMapStage =>
          val allPartitions = 0 until stage.numPartitions
          val filteredPartitions = allPartitions.filter { id => stage.outputLocs(id).isEmpty }
          (allPartitions, filteredPartitions)
        //在DAG Stage依赖关系中，最后的Stage为ResultStage 
        case stage: ResultStage =>
          val job = stage.resultOfJob.get
          val allPartitions = 0 until job.numPartitions
          val filteredPartitions = allPartitions.filter { id => !job.finished(id) }
          (allPartitions, filteredPartitions)
      }
    }
	// ... 设置累加变量
    if (stage.internalAccumulators.isEmpty || allPartitions == partitionsToCompute) {
      stage.resetInternalAccumulators()
    }  
      
    val properties = jobIdToActiveJob.get(stage.firstJobId).map(_.properties).orNull

    runningStages += stage

    //根据partitionsToCompute获取其优先位置PreferredLocations，使计算离数据最近
    val taskIdToLocations = try {
      stage match {
        case s: ShuffleMapStage =>
          partitionsToCompute.map { id => (id, getPreferredLocs(stage.rdd, id))}.toMap
        case s: ResultStage =>
          val job = s.resultOfJob.get
          partitionsToCompute.map { id =>
            val p = job.partitions(id)
            (id, getPreferredLocs(stage.rdd, p))
          }.toMap
      }
    } catch {
        //...
        return
    }

    stage.makeNewStageAttempt(partitionsToCompute.size, taskIdToLocations.values.toSeq)
    listenerBus.post(SparkListenerStageSubmitted(stage.latestInfo, properties))

    // 原子广播变量设置
    var taskBinary: Broadcast[Array[Byte]] = null
    try {
      // For ShuffleMapTask, serialize and broadcast (rdd, shuffleDep).
      // For ResultTask, serialize and broadcast (rdd, func).
      val taskBinaryBytes: Array[Byte] = stage match {
        case stage: ShuffleMapStage =>
          closureSerializer.serialize((stage.rdd, stage.shuffleDep): AnyRef).array()
        case stage: ResultStage =>
          closureSerializer.serialize((stage.rdd, stage.resultOfJob.get.func): AnyRef).array()
      }

      taskBinary = sc.broadcast(taskBinaryBytes)
    } catch {
      // In the case of a failure during serialization, abort the stage.
      // ...
    }

    //根据ShuffleMapTask或ResultTask，用于后期创建TaskSet
    val tasks: Seq[Task[_]] = try {
      stage match {
        case stage: ShuffleMapStage =>
          partitionsToCompute.map { id =>
            val locs = taskIdToLocations(id)
            val part = stage.rdd.partitions(id)
            //创建ShuffleMapTask
            new ShuffleMapTask(stage.id, stage.latestInfo.attemptId,
              taskBinary, part, locs, stage.internalAccumulators)
          }

        case stage: ResultStage =>
          val job = stage.resultOfJob.get
          partitionsToCompute.map { id =>
            val p: Int = job.partitions(id)
            val part = stage.rdd.partitions(p)
            val locs = taskIdToLocations(id)
             //创建ResultTask
            new ResultTask(stage.id, stage.latestInfo.attemptId,
              taskBinary, part, locs, id, stage.internalAccumulators)
          }
      }
    } catch {
      case NonFatal(e) =>
        abortStage(stage, s"Task creation failed: $e\n${e.getStackTraceString}", Some(e))
        runningStages -= stage
        return
    }

    if (tasks.size > 0) {
      stage.pendingTasks ++= tasks
      //重要！创建TaskSet并使用taskScheduler的submitTasks方法提交Stage
      taskScheduler.submitTasks(new TaskSet(
        tasks.toArray, stage.id, stage.latestInfo.attemptId, stage.firstJobId, properties))
      stage.latestInfo.submissionTime = Some(clock.getTimeMillis())
    } else {
      //提交完毕
      markStageAsFinished(stage, None)
    }
  }
```

### Task提交

上一节中TaskSet的生成说到最终stage被封装成TaskSet，使用taskScheduler.submitTasks提交，具体代码如下：

```scala
taskScheduler.submitTasks(new TaskSet(
        tasks.toArray, stage.id, stage.latestInfo.attemptId, stage.firstJobId, properties))
```

submitTasks方法定义在TaskScheduler Trait当中，目前TaskScheduler 只有一个子类TaskSchedulerImpl，其submitTasks方法源码如下：

```scala
//TaskSchedulerImpl类中的submitTasks方法
override def submitTasks(taskSet: TaskSet) {
    val tasks = taskSet.tasks
    this.synchronized {
      //创建TaskSetManager，TaskSetManager用于对TaskSet中的Task进行调度，包括跟踪Task的运行、Task失败重试等
      val manager = createTaskSetManager(taskSet, maxTaskFailures)
      val stage = taskSet.stageId
      //schedulableBuilder中添加TaskSetManager，用于完成所有TaskSet的调度，即整个Spark程序生成的DAG图对应Stage的TaskSet调度,schedulableBuilder中包含调度池等信息
      schedulableBuilder.addTaskSetManager(manager, manager.taskSet.properties)

      // 定时判断任务是否执行
      if (!isLocal && !hasReceivedTask) {
        starvationTimer.scheduleAtFixedRate(new TimerTask() {
          override def run() {
            if (!hasLaunchedTask) {
              logWarning("Initial job has not accepted any resources......")
            } else {
              this.cancel()
            }
          }
        }, STARVATION_TIMEOUT_MS, STARVATION_TIMEOUT_MS)
      }
      hasReceivedTask = true
    }
    //为Task分配运行资源
    backend.reviveOffers()
  }
```

将TaskSet 封装成TaskSetManager，通过schedulableBuilder去添加TaskSetManager到队列中，在Spark中，有两种形态

![img](https://img-blog.csdn.net/20170413163704600?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcmFpbnR1bmdsaQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center) 

1. FIFOSchedulableBuilder: 单一pool
2. FairSchedulableBuilder:   多个pool

在TaskSchedulerImpl在submitTasks添加TaskSetManager到pool后，调用了backend.reviveOffers();

==向driver的endpoint发送了reviveoffers的消息，Spark中的许多操作都是通过消息来传递的，哪怕DAGScheduler的线程和endpoint的线程都是同一个Driver进程==

### Task的分配

Netty 的dispatcher线程接受到revievoffers的消息后，CoarseGrainedSchedulerBackend

```
case ReviveOffers =>  
  makeOffers()  		
```

调用了makeoffers函数

``` scala
private def makeOffers() {  
      // Filter out executors under killing  
      val activeExecutors = executorDataMap.filterKeys(executorIsAlive)  
      val workOffers = activeExecutors.map { case (id, executorData) =>  
        new WorkerOffer(id, executorData.executorHost, executorData.freeCores)  
      }.toIndexedSeq  
      launchTasks(scheduler.resourceOffers(workOffers))  
    }  
```

makeOffers里进行了资源的调度，netty中接收所有的信息，同时也在CoarseGrainedSchedulerBackend中维护着executor的状态map:executorDataMap，executor的状态是executor主动汇报的。

通过scheduler.resourceOffers来进行task的资源分配到executor中

``` scala
def resourceOffers(offers: IndexedSeq[WorkerOffer]): Seq[Seq[TaskDescription]] = synchronized {  
   // Mark each slave as alive and remember its hostname  
   // Also track if new executor is added  
   var newExecAvail = false  
   for (o <- offers) {  
     if (!hostToExecutors.contains(o.host)) {  
       hostToExecutors(o.host) = new HashSet[String]()  
     }  
     if (!executorIdToRunningTaskIds.contains(o.executorId)) {  
       hostToExecutors(o.host) += o.executorId  
       executorAdded(o.executorId, o.host)  
       executorIdToHost(o.executorId) = o.host  
       executorIdToRunningTaskIds(o.executorId) = HashSet[Long]()  
       newExecAvail = true  
     }  
     for (rack <- getRackForHost(o.host)) {  
       hostsByRack.getOrElseUpdate(rack, new HashSet[String]()) += o.host  
     }  
   }  
  
   // Randomly shuffle offers to avoid always placing tasks on the same set of workers.  
   val shuffledOffers = Random.shuffle(offers)  
   // Build a list of tasks to assign to each worker.  
   val tasks = shuffledOffers.map(o => new ArrayBuffer[TaskDescription](o.cores))  
   val availableCpus = shuffledOffers.map(o => o.cores).toArray  
   val sortedTaskSets = rootPool.getSortedTaskSetQueue  
   for (taskSet <- sortedTaskSets) {  
     logDebug("parentName: %s, name: %s, runningTasks: %s".format(  
       taskSet.parent.name, taskSet.name, taskSet.runningTasks))  
     if (newExecAvail) {  
       taskSet.executorAdded()  
     }  
   }  
  
   // Take each TaskSet in our scheduling order, and then offer it each node in increasing order  
   // of locality levels so that it gets a chance to launch local tasks on all of them.  
   // NOTE: the preferredLocality order: PROCESS_LOCAL, NODE_LOCAL, NO_PREF, RACK_LOCAL, ANY  
   for (taskSet <- sortedTaskSets) {  
     var launchedAnyTask = false  
     var launchedTaskAtCurrentMaxLocality = false  
     for (currentMaxLocality <- taskSet.myLocalityLevels) {  
       do {  
         launchedTaskAtCurrentMaxLocality = resourceOfferSingleTaskSet(  
           taskSet, currentMaxLocality, shuffledOffers, availableCpus, tasks)  
         launchedAnyTask |= launchedTaskAtCurrentMaxLocality  
       } while (launchedTaskAtCurrentMaxLocality)  
     }  
     if (!launchedAnyTask) {  
       taskSet.abortIfCompletelyBlacklisted(hostToExecutors)  
     }  
   }  
  
   if (tasks.size > 0) {  
     hasLaunchedTask = true  
   }  
   return tasks  
 }  

```

1. 随机化了有效的executor的列表，为了均匀的分配
2. 获取池里（前面已经提过油两种池）的排号序的taskSetManager的队列
3. 对taskSetManager里面的task集合进行调度分配

#### taskSetManager队列的排序

这里的排序是对单个Pool里的taskSetManager进行排序，Spark有两种排序算法

``` scala
var taskSetSchedulingAlgorithm: SchedulingAlgorithm = {  
  schedulingMode match {  
    case SchedulingMode.FAIR =>  
      new FairSchedulingAlgorithm()  
    case SchedulingMode.FIFO =>  
      new FIFOSchedulingAlgorithm()  
    case _ =>  
      val msg = "Unsupported scheduling mode: $schedulingMode. Use FAIR or FIFO instead."  
      throw new IllegalArgumentException(msg)  
  }  
}  
```

#### 单个TaskSetManager的task调度

TaskSetManager 里保存了TaskSet，也就是DAGScheduler里生成的tasks的集合，在TaskSchedulerImpl.scala中进行了单个的TaskSetManager进行调度

在spark里，我们可以设置task所使用的cpu的数量，默认是1个，一个task任务在executor中是启动一个线程来执行的

通过计算每个executor的剩余资源，决定是否需要从tasksetmanager里分配出task.

在Spark中为了尽量分配任务到task所需的资源的本地,依据task里的preferredLocations所保存的需要资源的位置进行分配

1. 尽量分配到task到task所需资源相同的executor里执行，比如ExecutorCacheTaskLocation，HDFSCacheTaskLocation
2. 尽量分配到task里task所需资源相通的host里执行
3. task的数组从最后向前开始分配

分配完生成TaskDescription，里面记录着taskId, execId, task在数组的位置，和task的整个序列化的内容

#### driver启动Tasks

``` scala
private def launchTasks(tasks: Seq[Seq[TaskDescription]]) {  
      for (task <- tasks.flatten) {  
        val serializedTask = ser.serialize(task)  
        if (serializedTask.limit >= maxRpcMessageSize) {  
          scheduler.taskIdToTaskSetManager.get(task.taskId).foreach { taskSetMgr =>  
            try {  
              var msg = "..."
              taskSetMgr.abort(msg)  
            } catch {  
            }  
          }  
        }  
        else {  
          val executorData = executorDataMap(task.executorId)  
          executorData.freeCores -= scheduler.CPUS_PER_TASK  

          executorData.executorEndpoint.send(LaunchTask(new SerializableBuffer(serializedTask)))  
        }  
      }  
    }  
```

TaskDescription里面包含着executorId，而CoarseGrainedSchedulerBackend里有executor的信息，根据executorId获取到executor的通讯端口，发送LunchTask的信息。

这里有个RPC的消息的大小控制，如果序列化的task的内容超过了最大RPC的消息，这个任务会被丢弃

### Executor launch task

#### 启动任务

Driver向Executor发送了LaunchTask的消息，Executor接收到了LaunchTask的消息后，进行了任务的启动，在CoarseGrainedExecutorBackend.scala

``` scala
case LaunchTask(data) =>  
  if (executor == null) {  
    exitExecutor(1, "Received LaunchTask command but executor was null")  
  } else {  
    val taskDesc = ser.deserialize[TaskDescription](data.value)  
    logInfo("Got assigned task " + taskDesc.taskId)  
    executor.launchTask(this, taskId = taskDesc.taskId, attemptNumber = taskDesc.attemptNumber,  
      taskDesc.name, taskDesc.serializedTask)  
  }  
```

接收消息，反序列化了TaskDescription的对象

![img](https://img-blog.csdn.net/20170419094948831?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcmFpbnR1bmdsaQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center) 

在TaskDescription反序列化了taskId, executeId, name，index, attemptNumber, serializedTask属性，其中serializedTask是ByteBuffer。

Executor的launchTask方法

``` scala
def launchTask(  
     context: ExecutorBackend,  
     taskId: Long,  
     attemptNumber: Int,  
     taskName: String,  
     serializedTask: ByteBuffer): Unit = {  
   val tr = new TaskRunner(context, taskId = taskId, attemptNumber = attemptNumber, taskName,  
     serializedTask)  
   runningTasks.put(taskId, tr)  
   threadPool.execute(tr)  
 }  
```

方法中通过线程池中启动了线程运行TaskRunner的任务

`private val threadPool = ThreadUtils.newDaemonCachedThreadPool("Executor task launch worker")  `

关于线程池，在executor启动的是一个无固定大小线程数量限制的线程池，也就是说在executor的设计中，启动的任务数量是完全由Driver来管控

### 任务的运行

TaskDescription中的serializedTask是个bytebuffer, 里面的结构如下图所示：

![img](https://img-blog.csdn.net/20170419102729089?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcmFpbnR1bmdsaQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center) 

分别是task所依赖的文件的数量，文件的名字，时间戳，Jar的数量，Jar的名字，Jar的时间戳，属性，subBuffer是个bytebuffer 

#### 加载Jars文件

Driver所运行的class等包括依赖的Jar文件在Executor上并不存在，Executor首先要fetch所依赖的jars，也就是TaskDescription中serializedTask中的jar部分

在上面的结构描述中，jar相关的只是numJars,jarName,timestamp并没有jar的内容，也就是在LaunchTask里的消息中并不携带Jar的内容，原因也很容易理解，rpc的消息体必须简单高效

- timestamp:这是用于判断文件的时间戳，在相同文件名的情况下只有新的才需要重新fetch
- jarName: 这里的JarName是网络文件名：spark://192.168.121.101:37684/jars/spark-examples_2.11-2.1.0.jar

#### 运行task

![img](https://img-blog.csdn.net/20170419152654798?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcmFpbnR1bmdsaQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center) 

前面所提到的subBuffer实际上就是Task的序列化对象，通过反序列化可以获取到Driver生成的Task

在Executor.scala里的run方法中 

``` scala
/** 
   * Called by [[Executor]] to run this task. 
   * 被Executor调用以执行Task 
   * 
   * @param taskAttemptId an identifier for this task attempt that is unique within a SparkContext. 
   * @param attemptNumber how many times this task has been attempted (0 for the first attempt) 
   * @return the result of the task along with updates of Accumulators. 
   */  
  final def run(  
    taskAttemptId: Long,  
    attemptNumber: Int,  
    metricsSystem: MetricsSystem)  
  : (T, AccumulatorUpdates) = {  
    
    // 创建一个Task上下文实例：TaskContextImpl类型的context  
    context = new TaskContextImpl(  
      stageId,  
      partitionId,  
      taskAttemptId,  
      attemptNumber,  
      taskMemoryManager,  
      metricsSystem,  
      internalAccumulators,  
      runningLocally = false)  
        
    // 将context放入TaskContext的taskContext变量中  
    // taskContext变量为ThreadLocal[TaskContext]  
    TaskContext.setTaskContext(context)  
      
    // 设置主机名localHostName、内部累加器internalAccumulators等Metrics信息  
    context.taskMetrics.setHostname(Utils.localHostName())  
    context.taskMetrics.setAccumulatorsUpdater(context.collectInternalAccumulators)  
      
    // task线程为当前线程  
    taskThread = Thread.currentThread()  
      
    if (_killed) {// 如果需要杀死task，调用kill()方法，且调用的方式为不中断线程  
      kill(interruptThread = false)  
    }  
      
    try {  
      // 调用runTask()方法，传入Task上下文信息context，执行Task，并调用Task上下文的collectAccumulators()方法，收集累加器  
      (runTask(context), context.collectAccumulators())  
    } finally {  
      // 上下文标记Task完成  
      context.markTaskCompleted()  
        
      try {  
        Utils.tryLogNonFatalError {  
          // Release memory used by this thread for unrolling blocks  
          // 为unrolling块释放当前线程使用的内存  
          SparkEnv.get.blockManager.memoryStore.releaseUnrollMemoryForThisTask()  
          // Notify any tasks waiting for execution memory to be freed to wake up and try to  
          // acquire memory again. This makes impossible the scenario where a task sleeps forever  
          // because there are no other tasks left to notify it. Since this is safe to do but may  
          // not be strictly necessary, we should revisit whether we can remove this in the future.  
          val memoryManager = SparkEnv.get.memoryManager  
          memoryManager.synchronized { memoryManager.notifyAll() }  
        }  
      } finally {  
        // 释放TaskContext  
        TaskContext.unset()  
      }  
    }  
  }  
```

​	1、需要创建一个Task上下文实例，即TaskContextImpl类型的context，这个TaskContextImpl主要包括以下内容：Task所属Stage的stageId、Task对应数据分区的partitionId、Task执行的taskAttemptId、Task执行的序号attemptNumber、Task内存管理器taskMemoryManager、指标度量系统metricsSystem、内部累加器internalAccumulators、是否本地运行的标志位runningLocally（为false）；

​        2、将context放入TaskContext的taskContext变量中，这个taskContext变量为ThreadLocal[TaskContext]；

​        3、在任务上下文context中设置主机名localHostName、内部累加器internalAccumulators等Metrics信息；

​        4、设置task线程为当前线程；

​        5、如果需要杀死task，调用kill()方法，且调用的方式为不中断线程；

​        6、调用runTask()方法，传入Task上下文信息context，执行Task，并调用Task上下文的collectAccumulators()方法，收集累加器；

​        7、最后，任务上下文context标记Task完成，为unrolling块释放当前线程使用的内存，清楚任务上下文等。

##### runTask()

Task共分为两种类型，一个是最后一个Stage产生的ResultTask，另外一个是其parent Stage产生的ShuffleMapTask

ShuffleMapTask中的runTask()方法，定义如下：

``` scala
override def runTask(context: TaskContext): MapStatus = {  
    // Deserialize the RDD using the broadcast variable.  
    // 使用广播变量反序列化RDD  
      
    // 反序列化的起始时间  
    val deserializeStartTime = System.currentTimeMillis()  
      
    // 获得反序列化器closureSerializer  
    val ser = SparkEnv.get.closureSerializer.newInstance()  
      
    // 调用反序列化器closureSerializer的deserialize()进行RDD和ShuffleDependency的反序列化，数据来源于taskBinary  
    val (rdd, dep) = ser.deserialize[(RDD[_], ShuffleDependency[_, _, _])](  
      ByteBuffer.wrap(taskBinary.value), Thread.currentThread.getContextClassLoader)  
      
    // 计算Executor进行反序列化的时间  
    _executorDeserializeTime = System.currentTimeMillis() - deserializeStartTime  
  
    metrics = Some(context.taskMetrics)  
    var writer: ShuffleWriter[Any, Any] =   
    try {  
      // 获得shuffleManager  
      val manager = SparkEnv.get.shuffleManager  
        
      // 通过shuffleManager的getWriter()方法，获得shuffle的writer  
      // 启动的partitionId表示的是当前RDD的某个partition,也就是说write操作作用于partition之上  
      writer = manager.getWriter[Any, Any](dep.shuffleHandle, partitionId, context)  
        
      // 针对RDD中的分区<span style="font-family: Arial, Helvetica, sans-serif;">partition</span><span style="font-family: Arial, Helvetica, sans-serif;">，调用rdd的iterator()方法后，再调用writer的write()方法，写数据</span>  
      writer.write(rdd.iterator(partition, context).asInstanceOf[Iterator[_ <: Product2[Any, Any]]])  
        
      // 停止writer，并返回标志位  
      writer.stop(success = true).get  
    } catch {  
      case e: Exception =>  
        try {  
          if (writer != ) {  
            writer.stop(success = false)  
          }  
        } catch {  
          case e: Exception =>  
            log.debug("Could not stop writer", e)  
        }  
        throw e  
    }  
  }  
```

1. 通过使用广播变量反序列化得到RDD和ShuffleDependency：
   1. 获得反序列化的起始时间deserializeStartTime；
   2. 通过SparkEnv获得反序列化器ser；
   3. 调用反序列化器ser的deserialize()进行RDD和ShuffleDependency的反序列化，数据来源于taskBinary，得到rdd、dep；
   4. 计算Executor进行反序列化的时间_executorDeserializeTime；
2. 利用shuffleManager的writer进行数据的写入：
   1. 通过SparkEnv获得shuffleManager；
   2. 通过shuffleManager的getWriter()方法，获得shuffle的writer，其中的partitionId表示的是当前RDD的某个partition,也就是说write操作作用于partition之上；
   3. 针对RDD中的分区partition，调用rdd的iterator()方法后，再调用writer的write()方法，写数据；
   4. 停止writer，并返回标志位。

​          

ResultTask，其runTask()方法更简单

``` scala
override def runTask(context: TaskContext): U = {  
    // Deserialize the RDD and the func using the broadcast variables.  
      
    // 获取反序列化的起始时间  
    val deserializeStartTime = System.currentTimeMillis()  
      
    // 获取反序列化器  
    val ser = SparkEnv.get.closureSerializer.newInstance()  
      
    // 调用反序列化器ser的deserialize()方法，得到RDD和FUNC，数据来自taskBinary  
    val (rdd, func) = ser.deserialize[(RDD[T], (TaskContext, Iterator[T]) => U)](  
      ByteBuffer.wrap(taskBinary.value), Thread.currentThread.getContextClassLoader)  
      
    // 计算反序列化时间_executorDeserializeTime  
    _executorDeserializeTime = System.currentTimeMillis() - deserializeStartTime  
  
  
    metrics = Some(context.taskMetrics)  
      
    // 调针对RDD中的每个分区，迭代执行func方法，执行Task  
    func(context, rdd.iterator(partition, context))  
  }  
```

1. 通过SparkEnv获取反序列化器ser；
2. 调用反序列化器ser的deserialize()方法，得到RDD和FUNC，数据来自taskBinary；
3. 计算反序列化时间_executorDeserializeTime；
4. 调针对RDD中的每个分区，迭代执行func方法，执行Task。





