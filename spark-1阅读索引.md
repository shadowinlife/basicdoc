# Spark基本概念
Spark核心的三块分别是
* **SparkContext**的构成, 本质是spark如何构筑一个driver-worker的分布式引擎.在这个引擎中Driver部分告诉所有的worker要如何计算, 如何存储数据, 把数据送到哪里去等等metadata知识
* **BlockManager**也就是存储部分. Spark将数据存储到Memory或者Disk或者off heap也就是tachyon(Alluxio)分布式文件系统中. 将BlockID WorkerIP WorkerPort这些信息告诉Master, 这样全局的数据实际上是维护在Master节点的一个表里
* **JobScheduler**任务调度. 任务调度部分本质上是对RDD进行操作, Master通过对RDD的操作进行计算生成DAG图, 然后把DAG图切分成Stage, 每个Stage里有多个JOB, 这些JOB分发给worker去工作

>http://spark.apache.org/docs/latest/index.html\


# SPARK启动
```scala
class SparkEnv (
    val executorId: String,
    private[spark] val rpcEnv: RpcEnv,
    _actorSystem: ActorSystem, // TODO Remove actorSystem
    val serializer: Serializer,
    val closureSerializer: Serializer,
    val cacheManager: CacheManager,
    val mapOutputTracker: MapOutputTracker,
    val shuffleManager: ShuffleManager,
    val broadcastManager: BroadcastManager,
    val blockTransferService: BlockTransferService,
    val blockManager: BlockManager,
    val securityManager: SecurityManager,
    val sparkFilesDir: String,
    val metricsSystem: MetricsSystem,
    val memoryManager: MemoryManager,
    val outputCommitCoordinator: OutputCommitCoordinator,
    val conf: SparkConf)
```


* serializer 用计算资源换内存空间,存储相关
* cacheManager 存储相关
* mapOutputTracker **存储和计算的桥梁**
* shuffleManager **计算相关**
* broadcastManager 存储相关, 全局变量有两种, 文档里有说明
* blockTransferService 此服务其实是blockManager启动的子服务
* blockManager **存储核心服务**
* securityManager 安全先关, 第一个启动的, 所有服务都要验证权限
* metricssystem 统计服务, 和webui相关
* memoryManager 存储相关

# 阅读后续内容需要的前置知识
* 后续阅读针对有一定的spark基础的, 分析源码, 对spark的基本概念不了解需要仔细阅读官方文档. 按照官方文档跑通**所有**的官方自带例子.

* 在StandAlone模式下跑, 单机的话就跑一个Master一个Worker的StandAlone模式, 以增项对Akka和MR1.0的理解

* 需要阅读Hadoop2.6的一些文档, 主要是HDFS的基本使用, Yarn可以忽略.

* Spark最初的资源管理框架选择是Mesos, 在1.6.0以下版本选择Mesos可以弹性化的回收内存给同节点的其它executor使用.如果有能力, 最好了解以下Mesos的设计目标, 以及它和Yarn的区别.

* Netty和Event-Drvier模型 https://netty.io/wiki/user-guide-for-4.x.html\
spark主要使用Netty来实现worker和worker之间的数据块传输

* JavaNIO http://tutorials.jenkov.com/java-nio/index.html\
spark使用NIO一系列API来把数据刷到内存或者远端的机器上

* Akka和actor模型 https://akka.io/docs/\
Spark使用Akka来管理Driver和Executor, 实现运行时的通信, 维护一个Master-Slaver的拓扑结构分布式执行DAG任务




