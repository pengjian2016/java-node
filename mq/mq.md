### Kafka、RocketMQ、RabbitMQ

| 特性| Kafka | RocketMQ | RabbitMQ |
|-------|----------|----------|----------|
|  开发语言 |  scala  | java | erlang |
|  单机吞吐 | 10万级 | 10万级  |  万级  |
|  时效性  |   ms级以内 |     ms级     |  us级  |
|  可用性 |   非常高（分布式架构） |  非常高（分布式架构）  |   高（主从模式）  |
|  分布式事务 |   不支持 |  支持  |  不支持  |
|  优点 |   吞吐量非常高，设计架构天然支持可用性，大数据领域应用广泛 |  吞吐量和可用性也非常高，支持事务，功能比较完善，多用于交易系统  |  稳定、健壮、易用，社区活跃度高，适合中小型项目  |
|  缺点 |   分区数达到一定数量后吞吐量下降 |  客户端目前只支持java，其他尚未完善  |  吞吐量稍低一些，使用erlang开发，做二次开发扩展等较难  |


参考:

阿里测试组数据（文章比较老了，稍作参考）：[Kafka、RabbitMQ、RocketMQ等消息中间件的介绍和对比](https://blog.csdn.net/yunfeng482/article/details/72856762)

[高并发架构系列：Kafka、RocketMQ、RabbitMQ的优劣势比较](https://zhuanlan.zhihu.com/p/54450453)

### 消息丢失问题

消息丢失分为两种情况：

##### 1. 生产者丢失消息的情况

生产者发送消息后，可能由于网络等原因导致消息实际上并没有成功发送，从而造成丢失问题。

Kafka 消息丢失解决：

- 设置 acks = all（-1），表示所有副本都收到消息后才确认成功
- 设置 replication.factor > 1，要求每个 partition 必须有至少 2 个副本
- 设置 min.insync.replicas > 1，消息至少要被写入到 2 个副本才算是被成功发送

RocketMQ 消息丢失解决：

- 针对生产者到RocketMQ消息服务之间由于网络等原因丢失消息的可能，可以使用RocketMQ事务机制来发送消息，大致流程为：首先生产者发送half消息到RocketMQ中，此时消费者是无法消费half消息的，若half消息就发送失败了，则执行相应的回滚逻辑，half消息发送成功之后，且RocketMQ返回成功响应，则执行生产者的业务，如果生产者业务执行失败，则回滚，并通知RocketMQ删除half消息，如果生产者的业务执行成功，则通知RocketMQ commit half消息，让消费者可以消费这条数据，另外还要处理生产者长时间未发送提交或回滚的细节，这样可以保证生产者到RocketMQ消息服务之间不会丢失消息。

- RocketMQ消息服务丢失消息的情况，需要开启同步刷盘策略，这样同步刷盘成功才会返回确认消息。

RabbitMQ 消息丢失解决：

- 开启rabbitmq事务（channel.txSelect），发送消息时如果没有成功，生产者会收到异常报错，此时就可以回滚事务（channel.txRollback），然后重试发送消息；如果收到了消息，那么可以提交事务（channel.txCommit），但是使用事务机制的话会降低RabbitMQ的性能。
- 发送回执确认，开启confirm模式后，每次发送消息时都会分配一个唯一的id，如果消息成功发送到rabbitmq broker，会给你回传一个ack消息，告诉你说这个消息ok了，如果rabbitmq没能处理这个消息，会回调你一个nack接口，告诉你这个消息接收失败，你可以重试。而且你可以结合这个机制自己在内存里维护每个消息id的状态，如果超过一定时间还没接收到这个消息的回调，那么你可以重发，事务机制和cnofirm机制最大的不同在于，事务机制是同步的，你提交一个事务之后会阻塞在那儿，但是confirm机制是异步的，你发送个消息之后就可以发送下一个消息，然后那个消息rabbitmq接收了之后会异步回调你一个接口通知你这个消息接收到。
- RabbitMQ 服务丢失了消息，开启rabbitmq的持久化，写入消息的同时，持久化到磁盘中，即使消息服务挂了，再次重启仍然可以从磁盘中加载消息。设置持久化有两个步骤，第一个是创建queue的时候将其设置为持久化的，这样就可以保证rabbitmq持久化queue的元数据，但是不会持久化queue里的数据；第二个是发送消息的时候将消息的deliveryMode设置为2，就是将消息设置为持久化的，此时rabbitmq就会将消息持久化到磁盘上去。必须要同时设置这两个持久化才行，rabbitmq哪怕是挂了，再次重启，也会从磁盘上重启恢复queue，恢复这个queue里的数据。而且持久化可以跟生产者那边的confirm机制配合起来，只有消息被持久化到磁盘之后，才会通知生产者ack了，所以哪怕是在持久化到磁盘之前，rabbitmq挂了，数据丢了，生产者收不到ack，你也是可以自己重发的。若生产者那边的confirm机制未开启的情况下，哪怕是你给rabbitmq开启了持久化机制，也有一种可能，就是这个消息写到了rabbitmq中，但是还没来得及持久化到磁盘上，结果不巧，此时rabbitmq挂了，就会导致内存里的一点点数据会丢失。


> 注意：上表中说的是分布式事务，与这里的事务并非等价的概念，这里说的是消息发送到消息服务接收到消息之间的事务。RocketMQ事务消息（Transactional Message）是指应用本地事务和发送消息操作可以被定义到全局事务中，要么同时成功，要么同时失败。RocketMQ的事务消息提供类似 X/Open XA 的分布事务功能，通过事务消息能达到分布式事务的最终一致，以订单和扣库存这两个业务为例，它们分别为两个独立的系统，部署在不同的服务器上，当我们下单成功后，提交本地事务，同时库存服务完成口库存的逻辑，如果有一方失败，则另一方也要回滚，这就是两个系统之间事务一致性问题，RocketMQ支持的分布式事务就是解决这样的问题。当然并非说RabbitMQ和kafka完全做不了分布式事务，借助其他手段，加一些补偿机制等，也是可以完成的，只是相对复杂而已。

参考:

[RabbitMQ](https://github.com/apache/rocketmq/tree/master/docs/cn)

[RabbitMQ如何处理消息丢失](https://segmentfault.com/a/1190000019125512)

[分布式事务就是这么简单之RocketMQ解决方案](https://juejin.cn/post/6844903898214318087)

##### 2. 消费者丢失消息的情况


- kafka 消费方式分为自动提交和手动提交两种，自动提交的机制是根据一定的时间间隔，将收到的消息进行commit，commit过程和消费消息的过程是异步的，可能出现业务处理消息的过程中出现异常等，导致消息未处理成功，然而已经提交了。

解决这种情况可以将自动提交改为手动提交，当我们正真处理完业务后才进行提交。不过这样又会带来新的问题，假如我们的业务处理成功了，而在commit的时候提交失败，这就会导致消息重复消费问题。这个在下面的小节中处理。


- RocketMQ 在消费者接受到消息后，处理成功时才返回 ConsumeConcurrentlyStatus.CONSUME_SUCCESS 状态，只有返回成功状态才认为消费成功。

- RabbitMQ 与kafka类似，它也有自动确认机制，关闭自动ack，变成手动ack即可。

### 消息重复问题

由于某些原因导致一条消息被重复消费了多次，为了应对这种情况，需要我们消费者业务具有幂等性，即消费多次跟消费一次的结果是一致的，在生产者上保证消息不重复意义不大，仍然不能保证消息被重复消费的问题，所以最好在消费者端做处理，不管是kafka还是RocketMQ 解决方案都是类似的：

- 1. 每条消息有一个主键，利用数据库插入时主键冲突错误或者判断数据有相同的记录进行更新或丢弃等；
- 2. 将消费过的消息id存在redis中，写数据库之前判断redis中有就丢弃等；

### 消息顺序问题

有时候我们希望消息有顺序性，比如一笔订单，现有初始化消息才能有确认消息，这种情况如何保证呢？

- Kafka 中一个topic对应多个partition，每个partition下是有顺序的，一个partition只能被一个消费者消费，所以，可以将消息发送到相同的partition中，当然这只能保证局部顺序，如果希望全局顺序，需要保证：全局只有一个生产者，一个消费者（或者是一个消费线程），全局一个分区（partition）。

- RocketMQ 也是类似的原理，消息发送到相同的Queue（分区）中，每个Queue中的消息是有顺序的。

- RabbitMQ  把需要保证顺序的数据发到同一个Queue里

参考：

[RocketMQ顺序消费](https://www.cnblogs.com/qdhxhz/p/11134903.html)

[【RabbitMQ】如何保证消息的顺序性+解决消息积压+设计消息队列中间件](https://blog.csdn.net/qq_26545305/article/details/108203087)

### 消息堆积问题

生产者发送了大量的消息，消费者来不及处理，导致消息大量堆积。

生产环境中，我们不可能把生产者停掉等消费者消费完了消息之后再起，但是消息又再不断产生，紧急处理手段：

- 1. 降低生产者发送速度，比如限流等，当然这会导致一部分用户请求失败，体验不好；
- 2. 增加partition（当然也不能过多），增加消费者，每个消费者也使用多线程处理任务，如果还不能解决问题，则新建topic，让消息都发送到这个新的topic中，部署相应的消费者，后续通过写程序将那些过期或者未消费的消息抓取到过来；

事后分析消息堆积的原因，消费业务是否已经达到了瓶颈，各个环节有没有问题等。

### 消息重新消费问题

消息已经被正常消费过了，但由于某些原因（面试官就想这么问）需要重新消费消息，这个应该怎么办呢？

- kafka中，每个partition下的消息都有一个offset，如果要重新消费某些消息，可以指定offset，当然也可以用新的groupid从头开始消费等
- RocketMQ 中可以指定setConsumeTimestamp，消费时间，从这个时间点之后继续消费等；

参考：

[Kafka之重新消费数据](https://blog.csdn.net/usagoole/article/details/82813091)

[RocketMQ消费位置](https://my.oschina.net/mingxungu/blog/3083953)


##### kafka 

相关概念

Broker ： 安装Kafka服务的那台集群就是一个broker（broker的id要全局唯一）

Producer：消息的生产者，负责将数据写入到broker中（push方式）

Consumer：消息的消费者，负责从kafka中读取数据（pull）

Topic:主题，相当于是数据的一个分类，不同topic存放不同的数据

replication：副本，数据保存多少份（保证数据不丢）

partition：分区，是一个物理分区，一个分区就是一个文件，一个topic可以有一到多个分区，每一个分区都有自己的副本。

Consumer Group：消费者组，一个topic可以有多个消费者同时消费，多个消费者如果在一个消费者组中，那么它们不会消费相同分区下的消息，即一个分区只会被同一个组下的一个消息者消费，一个消费者却可以同时消费多个分区。

broker是服务，可以有多个topic，每个topic可有多个partion，不同的partion可以分布到不同的broker上，所以天然支持集群。

Broker集群中，会有一个leader（controller leader），负责管理整个集群中分区和副本的状态和选举partition leader

partion可以看作一个有序的队列，里面的数据是储存在硬盘中的，追加式的。partition的作用就是提供分布式的扩展，一个topic可以有许多partions，多个partition可以并行处理数据，所以可以处理相当量的数据。只有partition的leader才会进行读写操作，folower仅进行复制，客户端是感知不到的。


Kafka会在Zookeeper上针对每个Topic维护一个称为ISR（in-sync replica，已同步的副本）的集合，该集合中是一些分区的副本。只有当这些副本都跟Leader中的副本同步了之后，kafka才会认为消息已提交，并反馈给消息的生产者。如果这个集合有增减，kafka会更新zookeeper上的记录。如果某个分区的Leader不可用，Kafka就会从ISR集合中选择一个副本作为新的Leader

kafka集群中有2种leader，一种是broker的leader即controller leader，还有一种就是partition的leader：

Controller leader
当broker启动的时候，都会创建KafkaController对象，但是集群中只能有一个leader对外提供服务，这些每个节点上的KafkaController会在指定的zookeeper路径下创建临时节点，只有第一个成功创建的节点的KafkaController才可以成为leader，其余的都是follower。当leader故障后，所有的follower会收到通知，再次竞争在该路径下创建节点从而选举新的leader（关于zookeeper的创建与监听变化等，将在zookeeper那一章节介绍）

Partition leader 

由controller leader执行，从Zookeeper中读取当前分区的所有ISR(in-sync replicas)集合，调用配置的分区选择算法选择分区的leader

##### Kafka 消息是采用 Pull 模式，还是 Push 模式?

Kafka 遵循了一种大部分消息系统共同的传统的设计：producer(生产者) 将消息推送到 broker （push） ，consumer（消费者） 从 broker 拉取消息(pull)

##### Kafka零拷贝





参考：

[八年面试生涯，整理了一套Kafka面试题](https://juejin.cn/post/6844903889003610119)

[数据一致性 kafka 是保存副本 leader读写，follower 只备份 而 zookeeper是 leader 读写，follower负责读](https://www.cnblogs.com/aspirant/p/9179045.html)

[Kafka集群原理讲解及分区机制](https://blog.csdn.net/weixin_43866709/article/details/88989349)

[kafka集群 leader选举机制](https://my.oschina.net/u/3070368/blog/4338739)

##### RabbitMQ 集群

##### RocketMQ 集群