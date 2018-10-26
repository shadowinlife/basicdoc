# 1. 几个典型的队列系统和应用场景

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

# 3. Kafka的基本管理
## 设计理念
* 顺序度磁盘的性能和随机访问内存颗粒的性能类似, 因此核心问题是把队列系统的随机读变成顺序读

* 第一个解决的问题是 small I/O问题,文件系统的中位数大小大概4KB, Kafka在producer端缓存多个写请求到一个chunk, 在consumer端也是一次性读一个chunk的数据从而规避small IO 问题

* 第二个需要解决的问题是byte copy问题, KAFKA在Producer/Consumer/Broker上使用统一的二进制存储模型.对于kafka来说,这只是一个二进制流, 从而规避反复进行类型转换的overhead. 利用Linux内核中的sendfile接口, 可以把字节流从一个文件描述符直接写入到另外一个, 整个过程在内核空间完成, JVM不会涉及, 从而极大的减少了overhead. 详细https://www.ibm.com/developerworks/linux/library/j-zerocopy/

* 第三个解决带宽问题, 现代集群中, 最慢的是网卡/交换机, kakfa使用end-to-end compress, 用计算性能换取带宽. 这里需要注意, Kafka是支持LZ4压缩的. https://github.com/lz4/lz4

* 基于以上几点, kafka把producer的东西批量顺序写入磁盘, 然后批量顺序读出. 其中写过程, 存储过程, 读过程都是统一的二进制流, kafka在中间不会做任何类型处理等工作. 而且尽可能使用linux filesystem api来完成, 让数据在内核中流动.

## Producer
* 写过程尽可能是batch的, 默认情况下累积message到64KB或者累积时间到10ms触发写操作

* 默认的策略是向这个的topic的多个partition里轮流写入以保证balance, 不过用户也可以制定自己的策略, 比如只写到本机架

* 由于kafka的备份机制, 可以认为一个partition是由一个leader parition(主队列) 和 N 个 in-sync replica parition (ISR从队列)组成的, 在写入过程中. producer寻找到leader parition后会直接向它所在的broker机器写入.

* 写入时根据ack的设置来保证数据一致性符合预期. act=0情况下producer不会等borker server回复.  act=1情况下, procuer会等borker server完成leader partition写. act=all情况下, producer会等borker server告诉它leader parition和ISR都写成功了. 这个默认是1

## Consumer
* Consumer从broker中选择topic pull 数据过来, 每次尽可能的使用batch操作多拉一些数据, 同时在没有数据的时候, kakfa consumer支持 long polling操作等待新的数据到来

* Consumer需要面对两个问题, 一个是如何管理当前的读进度, 也就是offset. 另外一个是在parition失效时, 如何fail-over. 这两个问题都需要单独来讲解



# 4. Kafka的offset管理

# 5. Kafka的Replication机制

# 6. Kafka的扩容

## 增加节点
只要配置正确的集群名称, 唯一的boker-id, 启动后, 这个kafka的node就会被加入到kafka 集群中.但是在新的topic创建之前, 它是不会被实际使用!
> However these new servers will not automatically be assigned any data partitions, so unless partitions are moved to them they won't be doing any work until new topics are created. So usually when you add machines to your cluster you will want to migrate some existing data to these machines.

