redis 初识

#架构#

## sharding ##
redis 集群是主从式架构，数据分片是根据hash slot(哈希槽来分布)
总共有16384个哈希槽，所以理论上来说，集群的最大节点（master）
数量是16384个。一般推荐最大节点数量在1000个左右。

数据到shard的映射是根据传过来的key，CRC16生成值，然后对16834个哈希槽取模。目的就是数据能够均匀分布。
为。

没有mongo cluster 中mongos 角色。所有节点既要存储数据，也要存储
节点配置信息，比如某个hash槽的值对应在哪个节点上。客户端通信可以与
任意一个master节点连接。

redis是单线程，单进程的。所以可以拿来做分布式锁
## 一致性 ##
不是强一致性，因为从master-slave的复制是异步的

## 持久化 ##

RDB(redis database) 机制

隔一段时间去做一次copy,从内存到disk

AOF(append only file):
logs: 所有的写操作都记录下logs
需要将log写到磁盘中，因此代价比较大

SAVE:
强制redis server 去创建RDB snapshot

#数据类型#

**字符串（Strings）**
最多可以储存512字节的内容

**列表(Lists)**
按照插入顺序排序

    LPUSH list a

列表最多包含2^32 - 1个元素

**集合（Sets）**
无序字符串合集，最多包含2^32 - 1个元素

**有序集合(Sorted sets)**

**哈希（Hashed）**
key,value

    HSET username:a


#消息机制#

1. pub/sub

很简单，就是定义一个channel，然后将数据publish过去。
需要监听的客户端subscribe这个channel。就可以在onMessage方法中
处理监听到的消息了。

缺点就是一条消息会被多个监听的客户端都监听到，这样在水平扩展的应用服务中会有问题。

2. 列表，push pop

将数据push到链表中，然后pop出来，使用阻塞式BLPOP

