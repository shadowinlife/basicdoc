# 1. 几个典型的队列系统和应用场景
* Kafka是吞吐率最高的, 应用于大数据线消息队列, 比如持续吸入的日志, 电信数据, 金融数据. Kafka无法承载太多的端, 也就是不能有太多的consumer

* RocketMQ吞吐率略低于Kafka, 应用于互联网公司的消息注册和推送系统, 如向100万用户推送特定的新闻提示信息. 具体可以看阿里云的简介, 淘宝的系统

* Redis用于免锁, 高速的信息传递场景, 最典型的就是爬虫, 爬虫把自己爬到的url快速的写入到Redis队列中, 然后由后面的解析器去处理. Redis是免锁的, 而且速度极快, 微博主要用Redis来实现

* RabbitMQ ActiveMQ用于企业总线, 它们可以保证全局的信息顺序一致性, 这一点Kafka做不到. 同时保证系统掉电后不会丢数据, 这一点redis做不到. 是企业数据总线的唯一选择. 我们当前的企业总线选型是Kafka, 实际上无法保证操作幂等....没CTO是这样的啦....

# 2. Kafka的常用命令 (2.x.x)
* 社区的包里直接带一个zookeeper
```
in/zookeeper-server-start.sh config/zookeeper.properties
```
* 启动kafka
```
bin/kafka-server-start.sh config/server.properties
```
* 创建一套topic, 这里我们需要结合我们的实际生产环境说一下我们的partition数量和replica数量的选择
```
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 10 --topic osint-tumblr
```
* 创建一个消费者跟踪数据, 实际上会用中, 可以说我们的数据是on-the-fly的, 所以不需要from-beginning, 一般都是看流数据的质量
```
bin/kafka-console-consumer.sh --bootstrap-server kafka-server:9092 --topic osint-tumblr --from-beginning
```
* 扩容parition数量, 一般用于系统在线扩展的情况, 加入新的server后扩容会触发parition再分配, 从而利用上新加入的服务器
```shell
bin/kafka-topics.sh --zookeeper zk_host:port/chroot --alter --topic osint-tumblr --partitions 20
```

* 我们在生产中用kafka mirror maker把数据从美国/新加坡同步到AD
```shell
in/kafka-mirror-maker.sh
      --consumer.config consumer.properties
      --producer.config producer.properties --whitelist my-topic
```


* 在2.x.x中消费者group的offset是保存在kafka的文件系统中的, 可以用命令查看有哪些group
```shell
bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list
```
* 这些group当前的offset在哪里, partition对应的consumer有哪些等信息
```shell
bin/kafka-consumer-groups.sh --bootstrap-server kafka-server:9092 --describe --group my-group --members --verbose
```

# 3. Kafka概要
## 解决什么问题
* 顺序度磁盘的性能和随机访问内存颗粒的性能类似, 因此核心问题是把队列系统的随机读变成顺序读

* 第一个解决的问题是 small I/O问题,文件系统的中位数大小大概4KB, Kafka在producer端缓存多个写请求到一个chunk, 在consumer端也是一次性读一个chunk的数据从而规避small IO 问题

* 第二个需要解决的问题是byte copy问题, KAFKA在Producer/Consumer/Broker上使用统一的二进制存储模型.对于kafka来说,这只是一个二进制流, 从而规避反复进行类型转换的overhead. 利用Linux内核中的sendfile接口, 可以把字节流从一个文件描述符直接写入到另外一个, 整个过程在内核空间完成, JVM不会涉及, 从而极大的减少了overhead. 详细https://www.ibm.com/developerworks/linux/library/j-zerocopy/

* 第三个解决带宽问题, 现代集群中, 最慢的是网卡/交换机, kakfa使用end-to-end compress, 用计算性能换取带宽. 这里需要注意, Kafka是支持LZ4压缩的. https://github.com/lz4/lz4

* 基于以上几点, kafka把producer的东西批量顺序写入磁盘, 然后批量顺序读出. 其中写过程, 存储过程, 读过程都是统一的二进制流, kafka在中间不会做任何类型处理等工作. 而且尽可能使用linux filesystem api来完成, 让数据在内核中流动.

## Producer
* 写过程尽可能是batch的, 默认情况下累积message到64KB或者累积时间到10ms触发写操作

* 客户端的写操作和真正的写如到broker是分离异步的过程, 具体可以看下面的链接, 每个producer本质上是2个进程在处理.

* producer会周期性的更新cluster的metadata, 包括哪些node还或者, 每个node负责哪些parition等.具体可以看下面的链接

* 默认的策略是向这个的topic的多个partition里轮流写入以保证balance, 不过用户也可以制定自己的策略, 比如只写到本机架

* 由于kafka的备份机制, 可以认为一个partition是由一个leader parition(主队列) 和 N 个 in-sync replica parition (ISR从队列)组成的, 在写入过程中. producer寻找到leader parition后会直接向它所在的broker机器写入.

* 写入时根据ack的设置来保证数据一致性符合预期. act=0情况下producer不会等borker server回复.  act=1情况下, procuer会

等borker server完成leader partition写. act=all情况下, producer会需要borker server告诉它leader parition和ISR都写成功了. 这个默认是1


下面是一个翻译的非常烂的博客, 可以大体看看
> https://blog.csdn.net/szwandcj/article/details/76796459
https://blog.csdn.net/szwandcj/article/details/76796492?utm_source=blogxgwz2
https://blog.csdn.net/szwandcj/article/details/77460939?utm_source=blogxgwz0

大体结构可以看这个
>https://my.oschina.net/amethystic/blog/904568


## Consumer
* Consumer从broker中选择topic pull 数据过来, 每次尽可能的使用batch操作多拉一些数据, 同时在没有数据的时候, kakfa consumer支持 long polling操作等待新的数据到来

* Consumer需要面对两个问题, 一个是如何管理当前的读进度, 也就是offset. 另外一个是在parition失效时, 如何fail-over. 这两个问题需要看下面的链接

## 底层存储机制
>存储机制: https://yq.aliyun.com/ziliao/65771

>新版本同时把offset也存在本地

# 4. Kafka的集群管理和offset
* 0.9.1版本之前Kafka用zookeeper来管理offset, 每个parition存一份自己的offset在zookeeper里.这样的好处是ISR不用去同步offset到本地.
在0.9之后的版本, Kafka使用自己的broker来直接管理offset.

* Kafka的集群管理基本上就是靠offset来实现的, 当前有多少台broker是活的, 这些broker都在干什么, 是否有新增节点等等都是靠zookeeper实现. Broker认为zookeeper是不死的绝对一致的配置存储器, 有什么问题直接问它就好了.这里是zookeeper里面到底存了什么东西, 以及存储结构.
>https://cwiki.apache.org/confluence/display/KAFKA/Kafka+data+structures+in+Zookeeper


# 5. Kafka的Replication机制

## 一致性协议, 当一个broker挂掉了, 如何找到新的
Replication是用多备份来保证数据的安全性, HA的核心在写入时保证多个备份一致, 以及灾后保证fail-over. 
Kafka是采用一个leader后面多个ISR跟随的模式, 当leader失效后需要选举出一个新的leader, 它用的核心协议
>There are a rich variety of algorithms in this family including ZooKeeper's Zab, Raft, and Viewstamped Replication. The most similar academic publication we are aware of to Kafka's actual implementation is PacificA from Microsoft.
https://www.microsoft.com/en-us/research/publication/pacifica-replication-in-log-based-distributed-storage-systems/?from=http%3A%2F%2Fresearch.microsoft.com%2Fapps%2Fpubs%2Fdefault.aspx%3Fid%3D66814

选举结果被写入到zookeeper中, 然后consumer和producer就知道当前cluster的状态了.

## 涉及到的更多细节问题, 包括心跳是如何实现的, producer怎么知道它正在写的broker挂掉了等等.
>http://cloudurable.com/blog/kafka-architecture/index.html
需要把这些都读完

# 6. Kafka的扩容

## 增加节点
只要配置正确的集群名称, 唯一的boker-id, 启动后, 这个kafka的node就会被加入到kafka 集群中.但是在新的topic创建之前, 它是不会被实际使用!
> However these new servers will not automatically be assigned any data partitions, so unless partitions are moved to them they won't be doing any work until new topics are created. So usually when you add machines to your cluster you will want to migrate some existing data to these machines.

可以手动触发rebalance, 把一部分topic的parition放到这个新集群上去, 具体可以在命令工具中查找到.

# 7. Kafka的内存管理