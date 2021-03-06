## 1.主从方案（Sentinel 哨兵）

- [redis主从 + 哨兵 + VIP 的方式实现](./Redis主从高可用部署方案.md)
- [redis主从复制原理](https://juejin.im/post/5b67029c6fb9a04fa42fd592)

redis主从模式是最简单的一种集群方案配置起来也比较简单，它的特点主要有：

- 一个master可以拥有多个slave
- 多个slave链接同一个master，也可以链接其它slave
- 主从复制不会阻塞master,在同步数据时，master可以继续处理client请求.
- slave 配置为slave-read-only on需要升级为主节点或者写入配置文件中, 而不能在默认slave情况下直接设置master与slave断开后会检测心跳, 从新建立连接.
- 可以直接copy DUMP文件从新重启master，在Master为空以后，slave同步数据会抹掉全部数据.



## 2.分布式集群方案

### 2.1 客户端分片

这种方案将分片工作放在业务程序端，程序代码根据预先设置的路由规则，直接对多个Redis实例进行分布式访问。这样的好处是，不依赖于第三方分布式中间件，实现方法和代码都自己掌控，可随时调整，不用担心踩到坑。

这实际上是一种静态分片技术。Redis实例的增减，都得手工调整分片程序。基于此分片机制的开源产品，现在仍不多见。

这种分片机制的性能比代理式更好（少了一个中间分发环节）。但缺点是升级麻烦，对研发人员的个人依赖性强——需要有较强的程序开发能力做后盾。如果主力程序员离职，可能新的负责人，会选择重写一遍。

所以，这种方式下，可运维性较差。出现故障，定位和解决都得研发和运维配合着解决，故障时间变长。

这种方案，难以进行标准化运维，不太适合中小公司（除非有足够的DevOPS）。

### 2.2 代理分片

* [Codis分布式服务的解决方案](./Codis分布式服务的解决方案.md)

这种方案，将分片工作交给专门的代理程序来做。代理程序接收到来自业务程序的数据请求，根据路由规则，将这些请求分发给正确的Redis实例并返回给业务程序。

这种机制下，一般会选用第三方代理程序（而不是自己研发），因为后端有多个Redis实例，所以这类程序又称为分布式中间件。

这样的好处是，业务程序不用关心后端Redis实例，运维起来也方便。虽然会因此带来些性能损耗，但对于Redis这种内存读写型应用，相对而言是能容忍的。

这是我们推荐的集群实现方案。像基于该机制的开源产品Twemproxy，便是其中代表之一，应用非常广泛。

### 2.3 服务器分片（Redis Cluster）

* [Redis Cluster探索与思考](https://cloud.tencent.com/developer/article/1042654)

在这种机制下，没有中心节点（和代理模式的重要不同之处）。所以，一切开心和不开心的事情，都将基于此而展开。

Redis Cluster将所有Key映射到16384个Slot中，集群中每个Redis实例负责一部分，业务程序通过集成的Redis Cluster客户端进行操作。客户端可以向任一实例发出请求，如果所需数据不在该实例中，则该实例引导客户端自动去对应实例读写数据。

Redis Cluster的成员管理（节点名称、IP、端口、状态、角色）等，都通过节点之间两两通讯，定期交换并更新。

由此可见，这是一种非常“重”的方案。已经不是Redis单实例的“简单、可依赖”了。可能这也是延期多年之后，才近期发布的原因之一。

这令人想起一段历史。因为Memcache不支持持久化，所以有人写了一个Membase，后来改名叫Couchbase，说是支持Auto Rebalance，好几年了，至今都没多少家公司在使用。

这是个令人忧心忡忡的方案。为解决仲裁等集群管理的问题，Oracle RAC还会使用存储设备的一块空间。而Redis Cluster，是一种完全的去中心化……

本方案目前不推荐使用，从了解的情况来看，线上业务的实际应用也并不多见。



>  [redis集群方案总结](https://sealake.net/redisji-qun-fang-an-zong-jie/)



参考自：

https://www.jianshu.com/p/83ac498e1b2c

https://hoxis.github.io/redis-sentinel-ha.html