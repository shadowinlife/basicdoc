# 1. 项目整体结构
https://confluence.pegasus.ae/display/CB/OSINT+Ingestion+Pipeline+2.0

# 2. 项目规模
## 2.1. 数据量

1. Cold data on HDFS: 
        Total: 21.2 TB
        Key topics:
            Twitter:
                Compliance: 1.2 TB + 1.2 TB
                Decahose: 3.8 TB
                Firehose: 7.9 TB
                Powertrack: 5.5 TB + 1.5 TB
            DeepWeb:
                FlashpointReport: 50 MB
                FlashpointForumPost: 75.5 GB
            News:
                MoreoverNews: 59.6 GB
            Other Social Media:
                GooglePlus: 7.4 MB
                Instagram: 4.3 GB
                Tumblr: 40.6 MB
                Youtube: 23.1 GB
        Daily data size:
            Twitter:
                Compliance: 2.7 GB
                Decahose: 18 GB
                Firehose: 20 GB
                Powertrack: 19.5 GB
            DeepWeb:
                FlashpointForumPost: 50 MB
            News:
                MoreoverNews: 50 MB
## 2.2. 集群规模
OSINT 13TB，条数200Million

SIGINT 60TB，条数80billion

GOVINT 5TB，条数60million

EID 600G，体量40million

# 3. 数据库相关的问题
## 3.1. ElasticSearch

### 3.1.1. 乐观锁
> version
The update API uses the Elasticsearch’s versioning support internally to make sure the document doesn’t change during the update. You can use the version parameter to specify that the document should only be updated if its version matches the one specified.

其实就是update 一个document的时候带上version号.

### 3.1.2. 三层缓存
https://www.elastic.co/guide/en/elasticsearch/guide/current/filter-caching.html

我们当前的业务端写法中没有使用Filter关键字, 所以是不会触发这个缓存的.
这个缓存也是我们当前业务层形态唯一可以触发的缓存

https://www.elastic.co/guide/en/elasticsearch/reference/5.5/shard-request-cache.html

这个层级的cache对非常重的aggragation操作进行缓存,由于我们按照时间来划分index, 理论上老的index可以触发这个机制

https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-fielddata.html

这个层级的cache我们实际上可以参数调优

### 3.1.3. 并发锁机制
https://www.elastic.co/guide/en/elasticsearch/guide/current/concurrency-solutions.html






## 3.2 Hbase

###  3.2.1. key的选择

### 3.2.2. 并发问题

### 3.3.3. 性能调优问题

## 3.3 Redis

### 3.3.1. 用作缓存和存储

这个是业务层用的, 实际上在task-manager和ruler-service里也直接存储了很多规则.

由于redis是免锁的, 所以多个用户更新ruler-service里对datasift的爬取设置时, 不会出现冲突的情况.

### 3.3.2. 用作去重

我们曾经做过爬虫, 爬虫去重的时候根据url做primary key, 使用bloomfilter算法. 可以直接用在Redis里. 

https://redislabs.com/blog/rebloom-bloom-filter-datatype-redis/

RedisLabs直接提供一个bloomfilter的插件, 让redis原生支持这个算法, 最大可以使用512M个比特位. 实测了1亿个url在128M的空间上做去重, 冲突率是0


### 3.3.3. 用作分布式锁或者其它的一些中等规模并发场景

分布式系统加锁有很多解决方案, 最早是google把paxos算法实现成了chubby lock

https://static.googleusercontent.com/media/research.google.com/en//archive/chubby-osdi06.pdf

开源社区实现的zookeeper是一个魔改版本

https://cwiki.apache.org/confluence/display/ZOOKEEPER/Zab+vs.+Paxos

在实际应用中, 如果把zookeeper当成一个频繁使用的文件锁来用, 会遇到性能差的问题, 但是它有更完善的fail-over, 更加健壮.

对于健壮性要求比较低, 但是性能要求比较高的场景,  可以考虑这个

https://redis.io/topics/distlock

Redis的单线程特性, 导致它达到了隔离级别的串行化, 这个特性让很多操作"免锁". Redis默认设置是周期性刷硬盘的,  如果没刷下去会报错, 那个设置可以关掉, 变成一个纯内存数据库, 挂了就再也找不回来数据的那种.

## 3.4. HDFS


## 3.5. Zookeeper


# 4. 流处理相关的问题

## Beam

## Spark Stream

## Kafka
