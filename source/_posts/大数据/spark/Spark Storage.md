---
title: Spark Storage
date: 2018/5/12 15:30:25
category:
- 大数据
- spark
tag:
- spark
comments: true  
---

# Spark Storage

在Spark中提及最多的是RDD，而RDD所交互的数据是通过Storage来实现和管理

## Storage模块整体架构

### 存储层

在Spark里，单节点的Storage的管理是通过block来管理的，每个Block的存储可以在内存里或者在磁盘中，在BlockManager里既可以管理内存的存储，同时也管理硬盘的存储，存储的标识是通过块的ID来区分的。

![img](https://img-blog.csdn.net/20170320094910809?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcmFpbnR1bmdsaQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center) 

### 集群下的架构

在集群下Spark的Block的管理架构使用Master-Slave模式

- Master : 拥有所有block的具体信息（本地和Slave节点）
- Slave ： 通过master获取block的信息，并且汇报自己的信息

这里的Master并不是Spark集群中分配任务的Master，而是==**提交task的客户端Driver**==，这里并没有主备设计，因为Driver client是单点的，通常Driver client crash了，计算也没有结果了，**在Storage 的集群管理中Master是由driver承担。**

在Executor在运行task的时候，通过blockManager获取本地的block块，如果本地找不到，尝试通过master去获取远端的块

### Executor获取块内容的位置

![img](https://img-blog.csdn.net/20170320151259532?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcmFpbnR1bmdsaQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center) 

- 唯一的**blockID**: 

  **broadcast_0_piece0**
  请求Master获取该BlockID所在的 Location，也就是BlockManagerId的集合

  ```scala
  /** Get locations of the blockId from the driver */  
  def getLocations(blockId: BlockId): Seq[BlockManagerId] = {  
    driverEndpoint.askWithRetrySeq[BlockManagerId]  
  }  
  ```

- BlockManagerId

  **BlockManagerId(driver, 192.168.121.101, 55153, None)**

  - Host： executor/driver IP
  - Port:    executor/driver Port

  每一个executor, 和driver 都生成唯一的BlockManagerId

### Executor获取块的内容

通过获取的BlockManagerId的集合列表，顺序的从列表中取出一个拥有该Block的服务器，通过

``` scala
blockTransferService.fetchBlockSync(loc.host, loc.port, loc.executorId, blockId.toString).nioByteBuffer() 
```

同步的获取块的内容，如果该块不存在，则换下一个拥有该Block的服务器

### BlockManager注册

Driver 初始化SparkContext.init 的时候，会初始化BlockManager.initialize

``` scala
val idFromMaster = master.registerBlockManager(  
      id,  
      maxMemory,  
      slaveEndpoint)  
```

会通过master 注册BlockManager

```scala
def registerBlockManager(  
    blockManagerId: BlockManagerId,  
    maxMemSize: Long,  
    slaveEndpoint: RpcEndpointRef): BlockManagerId = {  
  logInfo(s"Registering BlockManager $blockManagerId")  
  val updatedId = driverEndpoint.askWithRetry[BlockManagerId](  
    RegisterBlockManager(blockManagerId, maxMemSize, slaveEndpoint))  
  logInfo(s"Registered BlockManager $updatedId")  
  updatedId  
}  
```

在BlockManagerMaster里，我们看到了endpoint是强制的driver，也就是默认是driver 是master

无论driver,还是executor都是初始化后BlockManager，发消息给driver master进行注册，唯一不同的是driver标识自己的ID是driver，而executor是按照executor id来标识自己的

### Driver Master的endpoint

无论driver还是executor 都会发送消息到Driver的Master，在Driver 和Executor里SparkEnv.create的时候会初始化BlockManagerMaster

``` scala
val blockManagerMaster = new BlockManagerMaster(registerOrLookupEndpoint(  
      BlockManagerMaster.DRIVER_ENDPOINT_NAME,  
      new BlockManagerMasterEndpoint(rpcEnv, isLocal, conf, listenerBus)),  
      conf, isDriver)  
```

注册一个lookup的endpoint

``` scala
def registerOrLookupEndpoint(  
        name: String, endpointCreator: => RpcEndpoint):  
      RpcEndpointRef = {  
      if (isDriver) {  
        logInfo("Registering " + name)  
        rpcEnv.setupEndpoint(name, endpointCreator)  
      } else {  
        RpcUtils.makeDriverRef(name, conf, rpcEnv)  
      }  
    }  
```

只有isDriver的时候才会setup一个rpc的endpoint，默认是netty的rpc环境，命名为：BlockManagerMaster

`spark://BlockManagerMaster@192.168.121.101:40978  `

所有的driver, executor都会向master 40978发消息

### Master和Executor消息格式

``` scala
override def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {  
    case RegisterBlockManager(blockManagerId, maxMemSize, slaveEndpoint) =>  
      context.reply(register(blockManagerId, maxMemSize, slaveEndpoint))  
  
    case _updateBlockInfo @  
        UpdateBlockInfo(blockManagerId, blockId, storageLevel, deserializedSize, size) =>  
      context.reply(updateBlockInfo(blockManagerId, blockId, storageLevel, deserializedSize, size))  
      listenerBus.post(SparkListenerBlockUpdated(BlockUpdatedInfo(_updateBlockInfo)))  
  
    case GetLocations(blockId) =>  
      context.reply(getLocations(blockId))  
  
    case GetLocationsMultipleBlockIds(blockIds) =>  
      context.reply(getLocationsMultipleBlockIds(blockIds))  
  
    case GetPeers(blockManagerId) =>  
      context.reply(getPeers(blockManagerId))  
  
    case GetExecutorEndpointRef(executorId) =>  
      context.reply(getExecutorEndpointRef(executorId))  
  
    case GetMemoryStatus =>  
      context.reply(memoryStatus)  
  
    case GetStorageStatus =>  
      context.reply(storageStatus)  
  
    case GetBlockStatus(blockId, askSlaves) =>  
      context.reply(blockStatus(blockId, askSlaves))  
  
    case GetMatchingBlockIds(filter, askSlaves) =>  
      context.reply(getMatchingBlockIds(filter, askSlaves))  
  
    case RemoveRdd(rddId) =>  
      context.reply(removeRdd(rddId))  
  
    case RemoveShuffle(shuffleId) =>  
      context.reply(removeShuffle(shuffleId))  
  
    case RemoveBroadcast(broadcastId, removeFromDriver) =>  
      context.reply(removeBroadcast(broadcastId, removeFromDriver))  
  
    case RemoveBlock(blockId) =>  
      removeBlockFromWorkers(blockId)  
      context.reply(true)  
  
    case RemoveExecutor(execId) =>  
      removeExecutor(execId)  
      context.reply(true)  
  
    case StopBlockManagerMaster =>  
      context.reply(true)  
      stop()  
  
    case BlockManagerHeartbeat(blockManagerId) =>  
      context.reply(heartbeatReceived(blockManagerId))  
  
    case HasCachedBlocks(executorId) =>  
      blockManagerIdByExecutor.get(executorId) match {  
        case Some(bm) =>  
          if (blockManagerInfo.contains(bm)) {  
            val bmInfo = blockManagerInfo(bm)  
            context.reply(bmInfo.cachedBlocks.nonEmpty)  
          } else {  
            context.reply(false)  
          }  
        case None => context.reply(false)  
      }  
  }  
```

### Master结构关系

![img](https://img-blog.csdn.net/20170320151224513?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcmFpbnR1bmdsaQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center) 

在Master上会保存每一个executor所对应的BlockManagerID和BlockManagerInfo，而在BlockManagerInfo中保存了每个block的状态

Executor通过心跳主动汇报自己的状态，Master更新EndPoint中Executor的状态

Executor 中的block的状态更新也会汇报给Master，只是跟新Master状态，但不会通知其他的Executor

在Executor和Master交互中是Executor主动推和获取数据的，Master只是管理executor的状态，以及Block的所在的Driver、Executor的位置及其状态，负载较小，Master没有考虑可用性，通常Master节点就是提交任务的Driver的节点。

## broadcast

### Spark BroadCast

Broadcast 简单来说就是将数据从一个节点复制到其他各个节点，常见用于数据复制到节点本地用于计算，Block既可以保存在内存中，也可以保存在磁盘中，当Executor节点本地没有数据，通过Driver去获取数据

在Broadcast中，Spark只是传递只读变量的内容，通常如果一个变量更新会涉及到多个节点的该变量的数据同步更新，为了保证数据一致性，Spark在broadcast 中只传递**不可修改的数据**。

Broadcast 只是细粒度化到executor? 在storage前面的文章中讨论过BlockID 是以executor和实际的block块组合的，executor 是执行submit的任务的子worker进程，随着任务的结束而结束，对executor里执行的子任务是同一进程运行，数据可以进程内直接共享（内存），所以BroadCast只需要细粒度化到executor就足够了

### TorrentBroadCast

Spark在老的版本1.2中有HttpBroadCast，但在2.1版本中就移除了，HttpBroadCast 中实现的原理是每个executor都是通过Driver来获取Data数据，这样很明显的加大了Driver的网络负载和压力，无法解决Driver的单点性能问题。

为了解决Driver的单点问题，Spark使用了Block Torrent的方式。

![img](https://img-blog.csdn.net/20170323113512591?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcmFpbnR1bmdsaQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center) \

1. Driver 初始化的时候，会知道有几个executor，以及Block, 最后在Driver端会生成block所对应的节点位置，初始化的时候因为executor没有数据，所有块的location都是Driver 
2. Executor 进行运算的时候，从BlockManager里的获取本地数据，如果本地数据不存在，然后从driver获取数据的位置
3. Driver里保存的块的位置只有Driver自己有，所以返回executer的位置列表只有driver
4. 通过块的传输通道从Driver里获取到数据
5. 获取数据后，使用BlockManager.putBytes保存数据
6. 在保存数据后同时汇报该Block的状态到Driver 
7. Driver跟新executor 的BlockManager的状态，并且把Executor的地址加入到该BlockID的地址集合中

#### 如何实现Torrent?

1. 为了避免Driver的单点问题，在上面的分析中每个executor如果本地不存在数据的时候，通过Driver获取了该BlockId的位置的集合，executor获取到BlockId的地址集合**随机化**后，优先找同主机的地址（这样可以走回环），然后从随机的地址集合按顺序取地址一个一个尝试去获取数据，因为随机化了地址，那么executor不只会从Driver去获取数据

2. BlockID 的随机化

   通常数据会被分为多个BlockID，取决于你设置的每个Block的大小

   在获取完整的BlockID块的时候，在Torrent的算法中，随机化了BlockID

在任务启动的时候，新启的executor都会同时从driver去获取数据，大家如果都是以相同的Block的顺序，基本上的每个Block数据对executor还是会从Driver去获取， 而BlockID的简单随机化就可以保证每个executor从driver获取到不同的块，当不同的executor在取获取其他块的时候就有机会从其他的executor上获取到，从而分散了对Driver的负载压力。

