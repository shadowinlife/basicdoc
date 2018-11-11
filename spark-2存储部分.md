# 1.spark是如何启动blockmanager的

所有的RDD底层是一系列的Block, 以及一个DAG图, 描述了如何把这些Block计算成新的Block最后返回给Driver. 这一部分讲解Block的存储和管理.

```scala
/** 
初始化一个blocktransferservice, 这里使用netty来做异步服务.
这个服务是用来在不同的worker之间交换数据用的底层服务.
*/
    val blockTransferService = new NettyBlockTransferService(conf, securityManager, numUsableCores)

/**
blockManger的管理节点, 本身是Akka里面的一个actor.
我们可以理解为所有的spark的工作节点上都有一个blockManagerMaster.
这些master们在akka里开圆桌会议, 所有woker上跑的master负责把自己这里的blockmanager的内容汇报给driver上的master.
所以可以看到在初始化时有个判断位isDriver
*/
    val blockManagerMaster = new BlockManagerMaster(registerOrLookupEndpoint(
      BlockManagerMaster.DRIVER_ENDPOINT_NAME,
      new BlockManagerMasterEndpoint(rpcEnv, isLocal, conf, listenerBus)),
      conf, isDriver)

/**
初始化blockManager, 这个里面实现了最底层的空间管理. 可以看到它需要用到序列化serializer, 需要用到内存管理部分memoryManager, 需要用到map的结果输出部分mapoutputtracker, 可能需要拿shuffle的结果shufflemanager
*/
    // NB: blockManager is not valid until initialize() is called later.
    val blockManager = new BlockManager(executorId, rpcEnv, blockManagerMaster,
      serializer, conf, memoryManager, mapOutputTracker, shuffleManager,
      blockTransferService, securityManager, numUsableCores)
```
# 2. BlockTransferService
```scala
/**
这里shuffleClient是一个abstract的class, 定义两个动作

*/
abstract class BlockTransferService extends ShuffleClient with Closeable with Logging {

  /**
   * Initialize the transfer service by giving it the BlockDataManager that can be used to fetch
   * local blocks or put local blocks.
   */
  def init(blockDataManager: BlockDataManager)

  /**
   * Tear down the transfer service.
   */
  def close(): Unit

  /**
   * Port number the service is listening on, available only after [[init]] is invoked.
   */
  def port: Int

  /**
   * Host name the service is listening on, available only after [[init]] is invoked.
   */
  def hostName: String

  /**
   * Fetch a sequence of blocks from a remote node asynchronously,
   * available only after [[init]] is invoked.
   *
   * Note that this API takes a sequence so the implementation can batch requests, and does not
   * return a future so the underlying implementation can invoke onBlockFetchSuccess as soon as
   * the data of a block is fetched, rather than waiting for all blocks to be fetched.
   */
  override def fetchBlocks(
      host: String,
      port: Int,
      execId: String,
      blockIds: Array[String],
      listener: BlockFetchingListener): Unit

  /**
   * Upload a single block to a remote node, available only after [[init]] is invoked.

   这里的ManagedBuffer的真正实现是NioManagedBuffer. 关于java的NewIO请自行阅读文档. 这里是对ByteBuffer进行了一次封装.用来管理Netty需要发送和取回的包.
   */
  def uploadBlock(
      hostname: String,
      port: Int,
      execId: String,
      blockId: BlockId,
      blockData: ManagedBuffer,
      level: StorageLevel): Future[Unit]

  /**
   * A special case of [[fetchBlocks]], as it fetches only one block and is blocking.
   *
   * It is also only available after [[init]] is invoked.
   */
  def fetchBlockSync(host: String, port: Int, execId: String, blockId: String): ManagedBuffer = {
    // A monitor for the thread to wait on.
    val result = Promise[ManagedBuffer]()
    fetchBlocks(host, port, execId, Array(blockId),
      new BlockFetchingListener {
        override def onBlockFetchFailure(blockId: String, exception: Throwable): Unit = {
          result.failure(exception)
        }
        override def onBlockFetchSuccess(blockId: String, data: ManagedBuffer): Unit = {
          val ret = ByteBuffer.allocate(data.size.toInt)
          ret.put(data.nioByteBuffer())
          ret.flip()
          result.success(new NioManagedBuffer(ret))
        }
      })

    Await.result(result.future, Duration.Inf)
  }

  /**
   * Upload a single block to a remote node, available only after [[init]] is invoked.
   *
   * This method is similar to [[uploadBlock]], except this one blocks the thread
   * until the upload finishes.
   */
  def uploadBlockSync(
      hostname: String,
      port: Int,
      execId: String,
      blockId: BlockId,
      blockData: ManagedBuffer,
      level: StorageLevel): Unit = {
    Await.result(uploadBlock(hostname, port, execId, blockId, blockData, level), Duration.Inf)
  }
}

# 3. BlockManagerMaster

## 3.1 初始化的参数
```scala
/**
RpcEndpointRef是一个线程安全的RPC句柄
isDriver 决定了这个BlockManagerMaster是启动在Mster点, 还是worker点
*/
class BlockManagerMaster(
    var driverEndpoint: RpcEndpointRef,
    conf: SparkConf,
    isDriver: Boolean)
  extends Loggin
```

## 3.2 RpcEndPointRef
下面给出几个核心的方法
```scala
/**
 * A reference for a remote [[RpcEndpoint]]. [[RpcEndpointRef]] is thread-safe.
 */
  private[spark] abstract class RpcEndpointRef(conf: SparkConf)
  extends Serializable with Logging {
 /**
   * Send a message to the corresponding [[RpcEndpoint.receiveAndReply)]] and return a [[Future]] to
   * receive the reply within the specified timeout.
   *
   * This method only sends the message once and never retries.
   */
  def ask[T: ClassTag](message: Any, timeout: RpcTimeout): Future[T]

  /**
   * Send a message to the corresponding [[RpcEndpoint.receiveAndReply)]] and return a [[Future]] to
   * receive the reply within a default timeout.
   *
   * This method only sends the message once and never retries.
   */
  def ask[T: ClassTag](message: Any): Future[T] = ask(message, defaultAskTimeout)

  /**
   * Send a message to the corresponding [[RpcEndpoint]] and get its result within a default
   * timeout, or throw a SparkException if this fails even after the default number of retries.
   * The default `timeout` will be used in every trial of calling `sendWithReply`. Because this
   * method retries, the message handling in the receiver side should be idempotent.
   *
   * Note: this is a blocking action which may cost a lot of time,  so don't call it in a message
   * loop of [[RpcEndpoint]].
   *
   * @param message the message to send
   * @tparam T type of the reply message
   * @return the reply message from the corresponding [[RpcEndpoint]]
   */
  def askWithRetry[T: ClassTag](message: Any): T = askWithRetry(message, defaultAskTimeout)

  /**
   * Send a message to the corresponding [[RpcEndpoint.receive]] and get its result within a
   * specified timeout, throw a SparkException if this fails even after the specified number of
   * retries. `timeout` will be used in every trial of calling `sendWithReply`. Because this method
   * retries, the message handling in the receiver side should be idempotent.
   *
   * Note: this is a blocking action which may cost a lot of time, so don't call it in a message
   * loop of [[RpcEndpoint]].
   *
   * @param message the message to send
   * @param timeout the timeout duration
   * @tparam T type of the reply message
   * @return the reply message from the corresponding [[RpcEndpoint]]
   */
  def askWithRetry[T: ClassTag](message: Any, timeout: RpcTimeout): T 
```
## 3.3 BlockManagerMaster的方法概要
BlockManagerMaster在HighLevel借助Akka实现了一个通信的抽象, 让各个worker的BlockManager能够相互通信, 同时维护整个存储系统的拓扑结构. 有那些Executor, 有哪些BroadCast变量等等

### 3.3.1 对通信的封装

基于3.2中说的RPCendpoint封装一个RPC调用, 用来和master节点进行通信
```scala
  /** Send a one-way message to the master endpoint, to which we expect it to reply with true. */
  private def tell(message: Any) {
    if (!driverEndpoint.askWithRetry[Boolean](message)) {
      throw new SparkException("BlockManagerMasterEndpoint returned false, expected true.")
    }
  }
  ```

### 3.3.2 和增加有关的方法
```scala
/** Register the BlockManager's id with the driver. */
  def registerBlockManager(
      blockManagerId: BlockManagerId, maxMemSize: Long, slaveEndpoint: RpcEndpointRef): Unit = {
    logInfo("Trying to register BlockManager")
    tell(RegisterBlockManager(blockManagerId, maxMemSize, slaveEndpoint))
    logInfo("Registered BlockManager")
  }
  /**
  把本地的Block信息更新到driver端, 包括Block的ID, 占用内存和磁盘的大小, 存储方式等. 这里存储方式有好多种, 启蒙文档里有, 就不复述了.
  */
  def updateBlockInfo(
      blockManagerId: BlockManagerId,
      blockId: BlockId,
      storageLevel: StorageLevel,
      memSize: Long,
      diskSize: Long,
      externalBlockStoreSize: Long): Boolean = {
    val res = driverEndpoint.askWithRetry[Boolean](
      UpdateBlockInfo(blockManagerId, blockId, storageLevel,
        memSize, diskSize, externalBlockStoreSize))
    logDebug(s"Updated info of block $blockId")
    res
  }

```

### 3.3.3 和查询有关的方法
*作为一个WORKER,我想知道某个文件块在哪台机器上*
```scala
   /** Get locations of the blockId from the driver */
  def getLocations(blockId: BlockId): Seq[BlockManagerId]

   /** Get locations of multiple blockIds from the driver */
  def getLocations(blockIds: Array[BlockId]): IndexedSeq[Seq[BlockManagerId]]
```
*作为一个worker, 我想知道当前都有哪些woker
```scala
/** Get ids of other nodes in the cluster from the driver */
  def getPeers(blockManagerId: BlockManagerId): Seq[BlockManagerId]

  def getExecutorEndpointRef(executorId: String): Option[RpcEndpointRef]
  ```

  *作为一个dirver, 我想知道当前worker们的工作状态*
  ```scala
  /**
   * Return the memory status for each block manager, in the form of a map from
   * the block manager's id to two long values. The first value is the maximum
   * amount of memory allocated for the block manager, while the second is the
   * amount of remaining memory.
   */
  def getMemoryStatus: Map[BlockManagerId, (Long, Long)]

  /**
   * Return the block's status on all block managers, if any. NOTE: This is a
   * potentially expensive operation and should only be used for testing.
   *
   * If askSlaves is true, this invokes the master to query each block manager for the most
   * updated block statuses. This is useful when the master is not informed of the given block
   * by all block managers.
   */
  def getBlockStatus(
      blockId: BlockId,
      askSlaves: Boolean = true): Map[BlockManagerId, BlockStatus]
  ```

### 3.3.4 和删除有关的方法
```scala

  /** 
  仅仅在driver端调用, 把挂掉的executor给去掉. executor是运行时概念, 是driver在worker上申请资源后运行的一个工作实例.

  Remove a dead executor from the driver endpoint. This is only called on the driver side. */
  def removeExecutor(execId: String) {
    tell(RemoveExecutor(execId))
    logInfo("Removed " + execId + " successfully in removeExecutor")
  }

   /**
   * Remove a block from the slaves that have it. This can only be used to remove
   * blocks that the driver knows about.
   */
  def removeBlock(blockId: BlockId)

   /** Remove all blocks belonging to the given RDD. */
  def removeRdd(rddId: Int, blocking: Boolean)

  /** Remove all blocks belonging to the given shuffle. */
  def removeShuffle(shuffleId: Int, blocking: Boolean)

  /** Remove all blocks belonging to the given broadcast. */
  def removeBroadcast(broadcastId: Long, removeFromMaster: Boolean, blocking: Boolean)
```

# 4. BlockManager概要
BlockManager聚焦在如何管理Executor本地的Block们, 实现底层的数据读取,下刷, 删除, 查询

## 4.1 BlockManagerId
用来注册一个唯一的BlockManager
```scala
class BlockManagerId private (
    private var executorId_ : String,
    private var host_ : String,
    private var port_ : Int)
  extends Externalizable
```

## 4.2 最简单的初始化
主要是启动shuffle相关的服务和上文提到的BlockTransferService用来和其它的woker节点交换数据.

启动后把自己挂到BlockManagerMaster下面
```scala
   /**
   * Initializes the BlockManager with the given appId. This is not performed in the constructor as
   * the appId may not be known at BlockManager instantiation time (in particular for the driver,
   * where it is only learned after registration with the TaskScheduler).
   *
   * This method initializes the BlockTransferService and ShuffleClient, registers with the
   * BlockManagerMaster, starts the BlockManagerWorker endpoint, and registers with a local shuffle
   * service if configured.
   */
  def initialize(appId: String): Unit
```

## 4.3 底层管理
```scala
  /** blockid被维护在一个TimeStampedHashMap里, 这是一个spark特别定制的hashmap, 在写入key的同时会写入一个timestamp, 并周期性的删除那些超时的key, 可以理解成blockmanager自己维护了一个简单版的redis在本地 **/
  private val blockInfo = new TimeStampedHashMap[BlockId, BlockInfo]

  // Actual storage of where blocks are kept
  private var externalBlockStoreInitialized = false
  // 所有的store都是BlockStore的子类

  // 管理内存里的block
  private[spark] val memoryStore = new MemoryStore(this, memoryManager)

  // 管理放在磁盘里的block管理器
  private[spark] val diskStore = new DiskStore(this, diskBlockManager)

  // 这个我没有读, 应该是放在Tachyon里的store管理器. 因为Tachyon到2.4.0都是experience的特性, 而且Tachyon本质上不存在了, 现在的项目是Alluxio. 这里
  private[spark] lazy val externalBlockStore: ExternalBlockStore 
```

## 4.4 BlockStore们都干了什么
因为已经用TimestampHashMap管理了BlockId, BlockStore们做的就是把这些BlockId实际刷到各种存储里.

这里涉及到三种子类,  分别是MemoryStore, DiskStore, TachyonStore.

其中MemoryStore的实现比较特别, 它采用了预申请的**占座**机制. 这个单独拿出来讲. 其它两种就是把字节流写入到文件系统的文件上, 文件名就是BlockId, 然后维护一个BlockId到文件系统路径的映射表而已.
```scala
private[spark] abstract class BlockStore(val blockManager: BlockManager) extends Logging {

  def putBytes(blockId: BlockId, bytes: ByteBuffer, level: StorageLevel): PutResult

  /**
   * Put in a block and, possibly, also return its content as either bytes or another Iterator.
   * This is used to efficiently write the values to multiple locations (e.g. for replication).
   *
   * @return a PutResult that contains the size of the data, as well as the values put if
   *         returnValues is true (if not, the result's data field can be null)
   */
  def putIterator(
    blockId: BlockId,
    values: Iterator[Any],
    level: StorageLevel,
    returnValues: Boolean): PutResult

  def putArray(
    blockId: BlockId,
    values: Array[Any],
    level: StorageLevel,
    returnValues: Boolean): PutResult

  /**
   * Return the size of a block in bytes.
   */
  def getSize(blockId: BlockId): Long

  def getBytes(blockId: BlockId): Option[ByteBuffer]

  def getValues(blockId: BlockId): Option[Iterator[Any]]

  /**
   * Remove a block, if it exists.
   * @param blockId the block to remove.
   * @return True if the block was found and removed, False otherwise.
   */
  def remove(blockId: BlockId): Boolean

  def contains(blockId: BlockId): Boolean

  def clear() { }
```

## 4.5 MemoryStore的机制
### 4.5.1 基本的组成
```scala
  private[spark] class MemoryStore(blockManager: BlockManager, maxMemory: Long)
    extends BlockStore(blockManager) {
    private val conf = blockManager.conf

    // 维护BlockId到内存区域的映射表
    private val entries = new LinkedHashMap[BlockId, MemoryEntry](32, 0.75f, true)
    @volatile private var currentMemory = 0L
    // Ensure only one thread is putting, and if necessary, dropping blocks at any given time
    private val accountingLock = new Object
    private val unrollMemoryMap = mutable.HashMap[Long, Long]()
    private val maxUnrollMemory: Long = {
        val unrollFraction = conf.getDouble("spark.storage.unrollFraction", 0.2)
        (maxMemory * unrollFraction).toLong
    }
private val unrollMemoryThreshold: Long =
    conf.getLong("spark.storage.unrollMemoryThreshold", 1024 * 1024)
def freeMemory: Long = maxMemory - currentMemory
```
### 4.5.2 占座机制