### Elasticsearch 集群

Elasticsearch 集群由一个或多个节点（Node）组成，集群中存在主节点（Master）和数据节点（data node）之分。

 **主节点（Master node）** ：负责管理集群范围内的所有变更，例如增加、删除索引，或者增加、删除节点等，而主节点并不需要涉及到文档级别的变更和搜索等操作，所以当集群只拥有一个主节点的情况下，即使流量的增加它也不会成为瓶颈

 **数据节点（data node）** ：负责保存数据、执行数据相关操作：CRUD、搜索、聚合等。数据节点对CPU、内存、I/O要求较高。一般情况下，数据读写流程只和数据节点交互，不会和主节点打交道

 **协调节点（Coordinating node）** ：协调节点的工作就是接收客户端请求，并且将收集数据所在分片的数据到自己这里，整合后返回给客户端。ElasticSearch集群中每个节点都可以作为协调节点（master、data都可以作为协调节点）

 **选主过程：** 

集群启动的第一件事是选择主节点，选主之后的流程由主节点触发。ES的选主算法是基于Bully算法的改进，主要思路是对节点ID排序，取ID值最大的节点作为Master，每个节点都运行这个流程。在实现上被分解为两步：先确定唯一的、大家公认的主节点，再想办法把最新的机器元数据复制到选举出的主节点上。基于节点ID排序的简单选举算法有三个附加约定条件：

1. 参选人数需要过半，达到quorum（多数）。
2. 得票数需过半。
3. 当探测到节点离开事件时，必须判断当前节点数是否过半。如果达不到quorum，则放弃Master身份，重新加入集群。如果不这么做，则可能产生双主，俗称脑裂。

集群并不知道自己共有多少个节点，quorum值从配置中读取，我们需要设置配置项：discovery.zen.minimum_master_nodes = master节点数/2 + 1。

#### Elasticsearch 如何实现分布式的呢？
都说Elasticsearch天然支持分布式，那如何做的呢？

ElasticSearch的索引上的数据是分散存储在分片上的，分片的数据组合起来才是完整的数据，一个节点上可以存在多个分片，一个分片就是一个Lucene实例，也就是说一个ElasticSearch节点上可能会运行很多的Lucene实例。通过分片的概念，数据和节点的绑定关系就被解耦了，因此ElasticSearch天然就是分布式的。扩容缩容只需要对分片进行迁移就可以了

ElasticSearch中的分片分为两种，一种是主分片（Primary Shard），另外一种是副本分片（Replica Shard）
主分片负责数据的写入操作，之后将变更同步给其他的副本分片。主分片和副本分片都可以处理数据检索请求。分片一方面提供了扩展ElasticSearch数据存储能力（可扩展性）、负载均衡的作用，还起到了数据备份的作用，当主分片所在节点出现问题时，可以使用副本分片进行主分片的替换，让整个集群继续正常提供服务，这也是Elasticsearch 高可用的原因。


参考：

[Elasticsearch集群启动流程](https://blog.csdn.net/qq_27639777/article/details/108170879)

[ElasticSearch工作机制](https://www.corgiboy.com/ElasticSearch/ElasticSearch%E5%B7%A5%E4%BD%9C%E6%9C%BA%E5%88%B6/)