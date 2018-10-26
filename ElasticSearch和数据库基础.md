ElasticSearch对我们来说相当于一种带有特殊索引(反向索引)的分布式数据库, 在复习ES的时候复习一些数据库的基本知识

# 1.CAP原理

* Consistency(一致性), 数据一致更新，所有数据变动都是同步的
* Availability(可用性), 好的响应性能
* Partition tolerance(分区容忍性) 可靠性

>定理：任何分布式系统只可同时满足二点，没法三者兼顾。

基本原理可以看这里 http://www.hollischuang.com/archives/666

这里涉及到一个数据库的基本概念"脑裂", 在保证分区容错性的情况下, 发生节点与节点之间的断连.  
产生脑裂, 这个时候要么左右大脑的数据不一致, 要么左右大脑因为不同步所以响应速度无限慢.  只能在C和A之间选一个

*在2012年的时候, CAP理论的作者伯克利的Brewer教授刊文, 表示当前工业界对CAP有一定的误解. CAP在牺牲极小可用性的前提下是可以全部满足的.*

*在实际的生产环境中, Amazon的DynamoDB 花钱版的Cassandra  淘宝的PolarDB 实际上是CAP都满足的*


# 2.BASE理论和NOSQL数据库选择
* Basically Available
* Soft State
* Eventual Consistency

HBase选的是CP, 放弃的是A, 如果HBase的某个Region Server挂掉了, 当时访问那个RS的所有的请求回返回网络响应错误, 这就不满足A了.

相对应的, 我们可以认为ES也是满足CP, 分布式的Redis也是满足CP


# 3.单机数据库的ACID和隔离级别

如果我们把所有的单机数据库, 比如MySQL看成只有一个点的分布式数据库.

那么本质上MySQL是一个放弃了P, 满足CA的"分布式数据库", 它满足ACID

* A: atomicity. 原子性
* C: consistency.  一致性
* I:: isolation. 隔离
* D: durability. 持久

MySQL的隔离级别
>https://www.cnblogs.com/huanongying/p/7021555.html

对于MySQL的更多问题, 比如InnoDB和MyISAM的区别等, 可以直接回复我用的是最新的7.0+版本, 就没有MyISAM选项. MyISAM适合无事务, 单线程的持续大量连续写入, 我们没有这个业务场景, 甚至Oracle都放弃了.  我们的多线程持续写入场景是向ES, HBase写入.



# 4.乐观锁, 悲观锁, ES使用的乐观锁

基本概念读这里
> https://www.jianshu.com/p/f5ff017db62a


乐观锁本质上是利用CPU的CAS操作的原子性, 保证每次只可能有一个进程修改内存空间中一个64位长度的long型的值. 这样我们可以把这个值看成一个version, 每次写入时都检查它, 如果和预期不符合, 就说明有其它的线程在修改它, 本次写入自动失败.

> version
The update API uses the Elasticsearch’s versioning support internally to make sure the document doesn’t change during the update. You can use the version parameter to specify that the document should only be updated if its version matches the one specified.

ES在执行update 一个document的时候带上version号, 如果version号和预期的不一样, 就会失败. 使用乐观锁时需要注意, 每次修改成功的同时对Version号+1s, 这样其它同时写的倒霉蛋就会因为version号不一致而失败掉.

# 5.ElasticSearch的缓存机制

## 5.1 第一层缓存保证filter操作是从bitmap里进行过滤

>https://www.elastic.co/guide/en/elasticsearch/guide/current/filter-caching.html

我们当前的业务端写法中没有使用Filter关键字, 所以是不会触发这个缓存的.
这个缓存也是我们当前业务层形态唯一可以触发的缓存

## 5.2 第二层缓存, 保证所有的节点在执行Aggragation操作时能把结果存在内存中, 直到这个节点的数据被更新.

>https://www.elastic.co/guide/en/elasticsearch/reference/5.5/shard-request-cache.html

这个层级的cache对非常重的aggragation操作进行缓存,由于我们按照时间来划分index, 理论上老的index可以触发这个机制

## 5.3 第三层缓存, 保证一部分Lucenne的数据不是在硬盘中的, 热数据在内存中.

>https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-fielddata.html

这个层级的cache我们实际上可以参数调优

# 6.ElasticSearch如何处理高并发加锁问题
>https://www.elastic.co/guide/en/elasticsearch/guide/current/concurrency-solutions.html

# 7.ElasticSearch的存储模型
## 英文原版
>https://blog.insightdatascience.com/anatomy-of-an-elasticsearch-cluster-part-i-7ac9a13b05db

>https://blog.insightdatascience.com/anatomy-of-an-elasticsearch-cluster-part-ii-6db4e821b571

>https://blog.insightdatascience.com/anatomy-of-an-elasticsearch-cluster-part-iii-8bb6ac84488d

对应HBse中用到的抽象数据结构是B+ Tree

在ES中数据以KD-Tree的形式组织在Lucenne文件中

## 中文翻译
http://www.cnblogs.com/zhangxiaoliu/p/7163761.html

*顺便吐槽一下, 国内做技术博客的是不是科技翻译毕业的*


# 8.Elasticsearch的数据倾斜, Rebalance, 伸缩存储节点

## 首先, 截止到我们用的版本, ES每个Index的分区数是在建立index时指定有多少个shard
>https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-create-index.html

## 其次, 这些shard可以自动的, 或者认为的指定它们被分配到哪些存储节点DataNode上. 

这里ES分配数据的逻辑和HDFS基本一致, 在建立集群的时候, 可以人工的给存储节点打上标签, 比如Rack1-NODE1  Rack2-Node2

在实际写Shard的时候可以指定, 我就要把某个shard写到rack1, 或者指定我就把数据迁移到某个IP上去.
>https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-allocation.html


## 第三, ES的弹性伸缩本质上是对shard到datanode进行分配
如果一开始建立的shard比datanode要少,那么新增节点也没有办法充分利用. 很多系统是一开始的时候就故意把shard数建立的多一点, 等到后边数据进来的多了新增节点, 就可以把一部分shard迁移过去.
>https://www.elastic.co/guide/en/elasticsearch/guide/current/scale.html

这个网址是官方对于横向扩展scale up的解释, 具体细节可以读一下.



# 9.ElasticSearch的调优和经验
Ebay公司今年1月份公布的调优经验
>https://www.ebayinc.com/stories/blogs/tech/elasticsearch-performance-tuning-practice-at-ebay/

如果把ES看成一个分布式计算系统,调优核心是在  End-to-End Latency 和 Throughput之间做选择, 要么端到端延迟低 (storm), 要么吞吐量高(spark stream). 根据数据量自动在两种业务形态之间选择是可能的, 比如淘宝魔改版本的Blink, 以及华为魔改版本的Flink
