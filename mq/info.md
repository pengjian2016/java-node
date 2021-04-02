
#### kafka 

基础概念

- Broker ： 安装Kafka服务的那台集群就是一个broker（broker的id要全局唯一）

- Producer：消息的生产者，负责将数据写入到broker中（push方式）

- Consumer：消息的消费者，负责从kafka中读取数据（pull）

- Topic:主题，相当于是数据的一个分类，不同topic存放不同的数据

- replication：副本，数据保存多少份（保证数据不丢）

- partition：分区，是一个物理分区，一个分区就是一个文件，一个topic可以有一到多个分区，每一个分区都有自己的副本。

- Consumer Group：消费者组，一个topic可以有多个消费者同时消费，多个消费者如果在一个消费者组中，那么它们不会消费相同分区下的消息，即一个分区只会被同一个组下的一个消息者消费，一个消费者却可以同时消费多个分区。

- offset：每个partition都由一系列有序的、不可变的消息组成，这些消息被连续的追加到partition中。partition中的每个消息都有一个连续的序列号叫做offset,用于partition唯一标识一条消息

#### kafka 集群

broker是服务，可以有多个topic，每个topic可有多个partion，不同的partion可以分布到不同的broker上，所以天然支持集群。

Broker集群中，会有一个leader（controller leader），负责管理整个集群中分区和副本的状态和选举partition leader

partion可以看作一个有序的队列，里面的数据是储存在硬盘中的，追加式的。partition的作用就是提供分布式的扩展，一个topic可以有许多partions，多个partition可以并行处理数据，所以可以处理相当量的数据。只有partition的leader才会进行读写操作，folower仅进行复制，客户端是感知不到的。

Kafka会在Zookeeper上针对每个Topic维护一个称为ISR（in-sync replica，已同步的副本）的集合，该集合中是一些分区的副本。只有当这些副本都跟Leader中的副本同步了之后，kafka才会认为消息已提交，并反馈给消息的生产者。如果这个集合有增减，kafka会更新zookeeper上的记录。如果某个分区的Leader不可用，Kafka就会从ISR集合中选择一个副本作为新的Leader

kafka集群中有2种leader，一种是broker的leader即controller leader，还有一种就是partition的leader：

Controller leader
当broker启动的时候，都会创建KafkaController对象，但是集群中只能有一个leader对外提供服务，每个节点上的KafkaController会在指定的zookeeper路径下创建临时节点，只有第一个成功创建的节点的KafkaController才可以成为leader，其余的都是follower。当leader故障后，所有的follower会收到通知，再次竞争在该路径下创建节点从而选举新的leader（关于zookeeper的临时节点创建与监听变化等，将在zookeeper那一章节介绍）

Partition leader 

由controller leader执行，从Zookeeper中读取当前分区的所有ISR(in-sync replicas)集合，调用配置的分区选择算法选择分区的leader

#### Kafka 消息是采用 Pull 模式，还是 Push 模式?

Kafka 遵循了一种大部分消息系统共同的传统的设计：producer(生产者) 将消息推送到 broker （push） ，consumer（消费者） 从 broker 拉取消息(pull)

#### Kafka高效文件存储的原因？

- Kafka把topic中一个parition大文件分成多个小文件段，通过多个小文件段，就容易定期清除或删除已经消费完文件，减少磁盘占用
- 通过索引信息可以快速定位message和确定response的最大大小
- 通过index元数据全部映射到memory，可以避免segment file的IO磁盘操作
- 通过索引文件稀疏存储，可以大幅降低index文件元数据占用空间大小

#### Kafka零拷贝

零拷贝并不是不需要拷贝，而是减少不必要的拷贝次数，也就是减少IO次数

传统IO过程：

比如：读取文件，再用socket发送出去

传统方式实现：

先读取、再发送，实际经过四次copy

1、第一次：将磁盘文件，读取到操作系统内核缓冲区；

2、第二次：将内核缓冲区的数据，copy到application应用程序的buffer；

3、第三步：将application应用程序buffer中的数据，copy到socket网络发送缓冲区(属于操作系统内核的缓冲区)；

4、第四次：将socket buffer的数据，copy到网卡，由网卡进行网络传输。

传统方式，读取磁盘文件并进行网络发送，经过的四次数据copy是非常繁琐的。实际IO读写，需要进行IO中断，需要CPU响应中断(带来上下文切换)，尽管后来引入DMA来接管CPU的中断请求，但四次copy是存在“不必要的拷贝”的。

重新思考传统IO方式，会注意到实际上并不需要第二个和第三个数据副本。应用程序除了缓存数据并将其传输回套接字缓冲区之外什么都不做。相反，数据可以直接从读缓冲区传输到套接字缓冲区。

显然，第二次和第三次数据copy 其实在这种场景下没有什么帮助反而带来开销，这也正是零拷贝出现的意义


零拷贝：只用将磁盘文件的数据复制到缓存中一次，然后将数据从页面缓存直接发送到网络中，减少了不必要的操作。


为什么Kafka这么快？

kafka作为MQ也好，作为存储层也好，无非是两个重要功能，一是Producer生产的数据存到broker，二是 Consumer从broker读取数据；我们把它简化成如下两个过程：

1、网络数据持久化到磁盘 (Producer 到 Broker)

2、磁盘文件通过网络发送（Broker 到 Consumer）

下面，先给出“kafka用了磁盘，还速度快”的结论

1、顺序读写

磁盘顺序读或写的速度400M/s，能够发挥磁盘最大的速度。随机读写，磁盘速度慢的时候十几到几百K/s。这就看出了差距。kafka将来自Producer的数据，顺序追加在partition，partition就是一个文件，以此实现顺序写入。Consumer从broker读取数据时，因为自带了偏移量，接着上次读取的位置继续读，以此实现顺序读。顺序读写，是kafka利用磁盘特性的一个重要体现。

2、零拷贝

数据直接复制到内核缓冲区，再发送到网络之中。



参考：

[八年面试生涯，整理了一套Kafka面试题](https://juejin.cn/post/6844903889003610119)

[数据一致性 kafka 是保存副本 leader读写，follower 只备份 而 zookeeper是 leader 读写，follower负责读](https://www.cnblogs.com/aspirant/p/9179045.html)

[Kafka集群原理讲解及分区机制](https://blog.csdn.net/weixin_43866709/article/details/88989349)

[kafka集群 leader选举机制](https://my.oschina.net/u/3070368/blog/4338739)

[Kafka文件存储机制那些事](https://tech.meituan.com/2015/01/13/kafka-fs-design-theory.html)

[Kafka零拷贝](https://blog.csdn.net/ljheee/article/details/99652448)

##### RocketMQ 高可用

RocketMQ分布式集群是通过Master和Slave的配合达到高可用性的。

Master和Slave的区别：在Broker的配置文件中，参数brokerId的值为0表明这个Broker是Master，大于0表明这个Broker是Slave，同时brokerRole参数也会说明这个Broker是Master还是Slave。

Master角色的Broker支持读和写，Slave角色的Broker仅支持读，也就是 Producer只能和Master角色的Broker连接写入消息；Consumer可以连接 Master角色的Broker，也可以连接Slave角色的Broker来读取消息。


消息生产的高可用：创建topic时，把topic的多个message queue创建在多个broker组上。这样当一个broker组的master不可用后，producer仍然可以给其他组的master发送消息。 rocketmq目前还不支持主从切换，需要手动切换

消息消费的高可用：consumer并不能配置从master读还是slave读。当master不可用或者繁忙的时候consumer会被自动切换到从slave读。这样当master出现故障后，consumer仍然可以从slave读，保证了消息消费的高可用

参考：

[RocketMQ高可用性机制](https://juejin.cn/post/6854573208772247565)

##### RabbitMQ 高可用
RabbitMQ 集群方案分为：

普通模式：
	- 1、RabbitMQ在多台服务器启动实例、每台服务器一个实例、当你创建queue时、queue（元数据+具体数据）只会落在一台RabbitMQ实例上、但是集群中每个实例都会同步queue的元数据（元数据：真实数据的描述如具体位置等）。
	- 2、当用户消费时如果连接的是另外一个实例，当前实例会根据同步的元数据找到具体的数据所在的实例从其上把具体数据拉过来消费。

缺点： 存放数据的queue的实例宕机后，会导致其它实例无法从该实例来拉取数据

镜像模式：在普通模式基础上做了改进，通过主节点把消息同步到每个节点的queue中去，如果主节点脱机了，则会从从节点中选举新的主节点，这就保证了高可用。

缺点： 性能开销太大，消息同步到所有的节点服务器会导致网络带宽压力和消耗很严重；

这种模式没有扩展性可言，如果你某个queue的负载很高，你加机器，新增的机器也包含了这个queue的所有数据，并没有办法线性扩展你的queue.
   
参考:

[如何保证消息队列（RabbitMQ）的高可用](https://segmentfault.com/a/1190000023008259)


以上三种MQ 实际工作中可能就用到其中的一种，选一个你熟悉的仔细研究，弄懂它的原理，这样面试的时候也就不用怕了。