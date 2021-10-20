# 主从模式、哨兵模式、集群模式（cluster）

redis 实现高可用的方式分为 主从模式、哨兵模式、集群模式（cluster）

### 1. 主从模式（又称为主从复制）
![输入图片说明](https://images.gitee.com/uploads/images/2021/1019/095158_4c4c32da_8076629.png "屏幕截图.png")

 **表现为1个主节点，多个从节点，主节点负责读和写，从节点只能读。** 


在主从架构中，master负责接收写请求，写操作成功后返回客户端OK，然后后将数据异步的方式发送给多个slaver进行数据同步，不过从redis 2.8开始，slave节点会周期性地确认自己每次复制的数据量。

当启动一个slave 节点的时候，它会发送一个PSYNC命令给master 节点。如果slave 节点是重新连接master 节点，那么master 节点仅仅会复制给slave部分缺少的数据; 否则如果是slave 节点第一次连接master 节点，那么会触发一次全量复制（full resynchronization）

开始全量复制的时候，master会启动一个后台线程，开始生成一份RDB快照文件，同时还会将从客户端收到的所有写命令缓存在内存（内存缓冲区）中。RDB文件生成完毕之后，master会将这个RDB发送给slave，slave会先写入本地磁盘，然后再从本地磁盘加载到内存中。然后master会将内存中缓存的写命令发送给slave，slave也会同步这些数据。

 **优点** 
1. 负载均衡：master能自动将数据同步到slave，可以进行读写分离，分担master的读压力，尤其是在写少读多的场景下，通过多个从节点分担读负载，可以大大提高 Redis 服务器的并发量
2. 故障恢复： 当主节点出现问题时，从节点仍然可以提供读服务，人工干预的情况下也可以实现快速的故障恢复
3. 高可用基石： 除了上述作用以外，主从复制还是哨兵和集群能够实施的 基础，因此说主从复制是 Redis 高可用的基础

 **缺点：** 
1. 主从模式中的主节点本质上还是单个redis实例，因此 主节点的 写 和 存储 性能仍受到单机的限制，主从模式只是提高了读的能力。

2. 主节点如果down机，系统无法提供写操作，建立在从节点上的链接仍然可以进行读操作，后续需要人工干预恢复主节点，或切换某个从节点为主节点（但是连接到原主节点的客户端需要修改配置，且在主节点down机之前发生的写操作，如果没有及时同步到从节点，这部分数据将会丢失或者从原主节点的AOF文件从手工恢复）


### 2. 哨兵模式
![输入图片说明](https://images.gitee.com/uploads/images/2021/1020/092942_4f652bad_8076629.png "屏幕截图.png")

主从模式中master挂掉后，系统无法提供写操作，没有人工干预的情况下不能自动选主。为了提高可用性，在主从模式的基础上增加哨兵(sentinel)，用来监控各节点情况，当master出现故障时，能自动将一个slave转换为master。哨兵节点是特殊的 Redis 节点，不存储数据。哨兵模式中，一般是一主多从加上3个及以上哨兵节点。

关于哨兵的功能描述：
- 监控（Monitoring）： 哨兵会不断地检查主节点和从节点是否运作正常。
   - 每个哨兵节点每10秒会向主节点和从节点发送info命令获取最新拓扑结构图，哨兵配置时只要配置对主节点的监控即可，通过向主节点发送info，获取从节点的信息，并当有新的从节点加入时可以马上感知到
   - 每个哨兵节点每隔2秒会向redis数据节点的指定频道上发送该哨兵节点对于主节点的判断以及当前哨兵节点的信息，同时每个哨兵节点也会订阅该频道，来了解其它哨兵节点的信息及对主节点的判断，其实就是通过消息publish和subscribe来完成的。
   -  每隔1秒每个哨兵会向主节点、从节点及其余哨兵节点发送一次ping命令做一次心跳检测，这个也是哨兵用来判断节点是否正常的重要依据。

- 自动故障转移（Automatic failover）： 当 主节点 不能正常工作时，哨兵会开始 自动故障转移操作，它会将失效主节点的其中一个 从节点升级为新的主节点，并让其他从节点改为复制新的主节点
- 配置提供者（Configuration provider）： 客户端在初始化时，通过连接哨兵来获得当前 Redis 服务的主节点地址
- 通知（Notification）： 哨兵可以将故障转移的结果发送给客户端

 **当master节点down机后进行选主流程：** 

##### 1. 哨兵认为master客观下线后，故障恢复的操作需要由选举出来的领头哨兵来执行，选举领头哨兵采用Raft算法
- 发现master下线的哨兵节点（我们称他为A）向每个哨兵发送命令，要求对方选自己为领头哨兵
- 如果目标哨兵节点没有选过其他人，则会同意选举A为领头哨兵
- 如果有超过一半（num(sentinel)/2+1）的哨兵同意选举A为领头，则A当选
- 如果有多个哨兵节点同时参选领头，此时有可能存在一轮投票无竞选者胜出，此时每个参选的节点等待一个随机时间后再次发起参选请求，进行下一轮投票竞选，直至选举出领头哨兵

##### 2. 选出领头哨兵后，领头者开始对系统进行故障恢复，从出现故障的master所在的系统中（哨兵可以监控多个master环境）的slave中挑选一个来当选新的master,选择规则如下
- 所有在线的slave中选择优先级最高的，优先级可以通过slave-priority配置
- 如果有多个最高优先级的slave，则选取复制偏移量最大（即复制越完整）的当选
- 如果以上条件都一样，选取id最小的slave
- 挑选出需要继任的slave后，领头哨兵向系统发送命令使其升格为master，然后再向其他slave发送命令接受新的master，最后更新数据。将已经停止的旧的master更新为slave节点，使其恢复服务后以slave的身份继续运行

如果哨兵挂掉了还能进行主从切换吗？

如果哨兵集群只有 2 个实例，此时一个哨兵要想成为 Leader，必须获得 2 票（超过半数），而不是 1 票，所以，如果有个哨兵挂掉了，那么此时的集群是无法进行主从库切换的。因此通常我们至少会配置 3 个哨兵实例。

 **优点：** 
1. 主从模式的优点它都有
2. 在主从模式的基础上增加了哨兵，实现了master节点的自动切换，系统可用性更高
 **缺点：** 
1. 同样也继承了主从模式难以在线扩容的缺点，Redis的容量受限于单机配置
2. 需要额外的资源来启动哨兵服务，实现相对复杂一点

### 3. 集群模式（又称为cluster模式）

哨兵模式解决了主节点自动切换的问题，实现了高可用性，但是单个节点的存储能力是有上限，访问能力是有上限的。Redis Cluster 集群模式具有高可用、可扩展性、分布式、容错等特性

参考：

[一文掌握Redis主从复制、哨兵、Cluster三种集群模式）](https://juejin.cn/post/6844904097116585991)

[《「面试突击」— Redis篇》-- Redis的主从复制？哨兵机制？](https://juejin.cn/post/6844904080918347784)

[Redis单机、主从、哨兵、集群模式](https://www.jianshu.com/p/d5b144349b70)