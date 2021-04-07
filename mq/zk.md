# zookeeper

它是一个分布式协调服务，可以用来实现：

- 命名服务：根据指定名字来获取资源、服务的地址、提供者信息等，例如：某个接口B部署了多台服务有多个ip，它可以在zookeeper中创建一个统一的节点在这个节点下维护一个列表，当B服务增加或减少等，在zookeeper中的列表也会发生变化，同时调用B接口的服务只需要在调用前从zookeeper对外提供的统一名称获取列表即可。
- 配置管理：提供统一的系统配置管理。
- 分布式锁：分布式服务下某个时刻只能有一个服务能执行某个业务。
- 集群管理：服务注册与发现中心，负载均衡（轮询服务注册表，尽可能将服务请求均匀分配到所有注册有效的服务器上），服务健康监控等。

# zookeeper 数据结构

- 层次化的目录结构，命名符合常规文件系统规范
- 每个节点在zookeeper中叫做znode,并且其有一个唯一的路径标识
- 节点Znode可以包含数据(只能存储很小量的数据，<1M;最好是1k字节以内)和子节点（但是ephemeral (临时节点)类型的节点不能有子节点)
- 客户端应用可以在节点上设置监视器

![数据结构](https://images.gitee.com/uploads/images/2021/0406/155016_ffbd5821_8076629.png "数据结构.png")

znode节点类型：

- 持久节点（persistent node）节点会被持久化
- 临时节点（ephemeral node），客户端断开连接后，ZooKeeper会自动删除临时节点
- 持久顺序节点（persistent_sequential），每次创建顺序节点时，ZooKeeper都会在路径后面自动添加上10位的数字，从1开始，最大是2147483647 （2^32-1）
- 临时顺序节点 （ephemeral_sequential），有顺序的临时节点

znode 属性：

```
[zk: localhost:2181(CONNECTED) 22] get /test
test
cZxid = 0x23c
ctime = Tue Apr 06 08:03:10 UTC 2021
mZxid = 0x23c
mtime = Tue Apr 06 08:03:10 UTC 2021
pZxid = 0x240
cversion = 4
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 4
numChildren = 4
```
- cZxid：创建时的事务id，每次的变化都会产生一个集群全局的唯一的事务id， Zxid（ZooKeeper Transaction Id），由Zookeeper的leader实例维护。
- ctime：创建时间
- mZxid：修改的事务标识，每次修改操作（set）后都会更新mZxid和mtime
- mtime：修改时间
- pZxid：子节点最后更新的事务标识，每个子节点更新变化时都会更新这个值
- cversion: 当子节点有变化时，版本号就会增加1
- dataVersion：节点数据发送变化时，版本号就会增加1
- aclVersion：节点acl(Access Control Lists)版本号
- ephemeralOwner：：当前节点是临时节点（ephemeral node ）时，这个ephemeralOwner的值是客户端持有的session id。
- dataLength：节点存储的数据长度
- numChildren：子节点的个数



# Paxos 算法

Paxos算法是基于消息传递且具有高度容错特性的共识（consensus）算法，是目前公认的解决分布式一致性问题最有效的算法之一。

需要注意的是，Paxos常被误称为“一致性算法”。但是“一致性（consistency）”和“共识（consensus）”并不是同一个概念。Paxos是一个共识（consensus）算法

由于Paxos算法晦涩难懂，这里不做过多深入研究推导等，有兴趣的朋友可以读一下参考列表中的文章。

在Paxos算法中，有Proposer（可以提出提案-实际的value值）、Acceptor（可以接受提案）以及Learner（学习被批准的提案）三种角色，当然一个进程可能同时充当多种角色，比如一个进程可能既是Proposer又是Acceptor又是Learner。

Paxos算法通过一个决议分为两个阶段：

#### 1.prepare阶段：
- Proposer选择一个提案编号N（Proposer生成全局唯一且递增的Proposal ID (可使用时间戳加Server ID)），然后向半数以上的Acceptor发送编号为N的Prepare请求
- 如果一个Acceptor收到一个编号为N的Prepare请求，且N大于该Acceptor已经响应过的所有Prepare请求的编号，那么它就会将它已经接受过的编号最大的提案（如果有的话）作为响应反馈给Proposer，同时该Acceptor承诺不再接受任何编号小于N的提案。

#### 2.批准阶段：
- 如果Proposer收到半数以上Acceptor对其发出的编号为N的Prepare请求的响应，那么它就会发送一个包括编号和提案value即[N,V]的Accept请求给半数以上的Acceptor。注意：V就是收到的响应中编号最大的提案的value，如果响应中不包含任何提案，那么V就由Proposer自己决定。
-  如果Acceptor收到一个针对编号为N的提案的Accept请求，只要该Acceptor没有对编号大于N的Prepare请求做出过响应，它就接受该提案

这个过程在任何时候中断都可以保证正确性。例如如果一个proposer发现已经有其他proposers提出了编号更高的提案，则有必要中断这个过程。因此为了优化，在上述prepare过程中，如果一个acceptor发现存在一个更高编号的提案，则需要通知proposer，提醒其中断这次提案。

#### Paxos 算法的活锁问题

活锁指的是任务或者执行者没有被阻塞，由于某些条件没有满足，导致一直重复尝试—失败—尝试—失败的过程。处于活锁的实体是在不断的改变状态，活锁有可能自行解开。

假如编号1的提案prepare阶段已经通过（超过半数），在accept阶段发送前，或者发送中消息还没有到达其他Server时，有新的编号2提案，比编号1提案先到达，并且超过半数，（此时编号1提案是accpet阶段，编号2是prepare阶段）此时编号1的提案在到达时肯定收不到半数accpet的，而编号2的提案在accept阶段发送前，又有新的提案编号3在prepare阶段先到达并且超过半数通过…以此类推，永远也不会决定出一个提案通过。

# Multi-Paxos算法

原始的Paxos算法（Basic Paxos）只能对一个值形成决议，决议的形成至少需要两次网络来回，在高并发情况下可能需要更多的网络来回，极端情况下甚至可能形成活锁。如果想连续确定多个值，Basic Paxos搞不定了。因此Basic Paxos几乎只是用来做理论研究，并不直接应用在实际工程中。

实际应用中几乎都需要连续确定多个值，而且希望能有更高的效率。Multi-Paxos正是为解决此问题而提出。Multi-Paxos基于Basic Paxos做了两点改进：

- 针对每一个要确定的值，运行一次Paxos算法实例（Instance），形成决议。每一个Paxos实例使用唯一的Instance ID标识。
- 在所有Proposers中选举一个Leader，由Leader唯一地提交Proposal给Acceptors进行表决。这样没有Proposer竞争，解决了活锁问题。在系统中仅有一个Leader进行Value提交的情况下，Prepare阶段就可以跳过，从而将两阶段变为一阶段，提高效率。

Multi-Paxos首先需要选举Leader，Leader的确定也是一次决议的形成，所以可执行一次Basic Paxos实例来选举出一个Leader。选出Leader之后只能由Leader提交Proposal，在Leader宕机之后服务临时不可用，需要重新选举Leader继续服务。在系统中仅有一个Leader进行Proposal提交的情况下，Prepare阶段可以跳过。

Multi-Paxos通过改变Prepare阶段的作用范围至后面Leader提交的所有实例，从而使得Leader的连续提交只需要执行一次Prepare阶段，后续只需要执行Accept阶段，将两阶段变为一阶段，提高了效率。为了区分连续提交的多个实例，每个实例使用一个Instance ID标识，Instance ID由Leader本地递增生成即可。


### Zookeeper 相关概念

集群中的三类角色：

- Leader：一个ZooKeeper集群同一时间只会有一个实际工作的Leader，它会发起并维护与各Follwer及Observer间的心跳。所有的写操作必须要通过Leader完成再由Leader将写操作广播给其它服务器
- Follower: 一个ZooKeeper集群可能同时存在多个Follower，它会响应Leader的心跳。Follower可直接处理并返回客户端的读请求，同时会将写请求转发给Leader处理，并且负责在Leader处理写请求时对请求进行投票。
- Observer 角色与Follower类似，只是不参与 Leader 选举投票。

通过Leader 进行写请求，步骤：
- 客户端向Leader发起写请求；
- Leader将写请求以Proposal（提议）的形式发给所有Follower并等待ACK；
- Follower收到Leader的Proposal（提议）后返回ACK
- Leader得到过半数的ACK（Leader对自己默认有一个ACK）后向所有的Follower和Observer发送Commmit（提交事务）
- Leader将处理结果返回给客户端

Leader并不需要得到Observer的ACK，即Observer无投票权，Leader不需要得到所有Follower的ACK，只要收到过半的ACK即可，同时Leader本身对自己有一个ACK，Observer虽然无投票权，但仍须同步Leader的数据，从而在处理读请求时可以返回尽可能新的数据

通过Follower/Observer进行写请求：
- Follower/Observer均可接受写请求，但不能直接处理，而需要将写请求转发给Leader处理，除了多一步请求转发之外，与通过Leader 进行写请求无需别。

读操作：
- Leader/Follower/Observer都可直接处理读请求，从本地内存中读取数据并返回给客户端即可。

集群中服务的状态：
- LOOKING 选举状态，不确定Leader状态。该状态下的服务器认为当前集群中没有Leader，会发起Leader选举。
- FOLLOWING 跟随者状态。表明当前服务器角色是Follower，并且它知道Leader是谁
- LEADING 领导者状态。表明当前服务器角色是Leader，它会维护与Follower间的心跳。
- OBSERVING 观察者状态。表明当前服务器角色是Observer，与Folower唯一的不同在于不参与选举，也不参与集群写操作时的投票。

# Zab 协议

zab（Zookeeper Atomic Broadcast） zookeeper 原子广播协议。 ZAB本质上就是Paxos的一种简化形式。它是一种特别为ZooKeeper设计的崩溃可恢复的原子消息广播算法

当整个服务框架在启动过程中，或是当Leader服务器出现网络中断崩溃退出与重启等异常情况时，ZAB就会进入恢复模式并选举产生新的Leader服务器。

当选举产生了新的Leader服务器，同时集群中已经有过半的机器与该Leader服务器完成了状态同步之后，ZAB协议就会退出崩溃恢复模式，进入消息广播模式。

当有新的服务器加入到集群中去，如果此时集群中已经存在一个Leader服务器在负责进行消息广播，那么新加入的服务器会自动进入数据恢复模式，找到Leader服务器，并与其进行数据同步，然后一起参与到消息广播流程中去。

对以上的状态描述分为三种模式：恢复模式、广播模式、同步模式

当然这三种模式并没有十分明显的界线，它们相互交织在一起

#### 崩溃恢复模式

当集群正在启动过程中，或 Leader 崩溃后，集群就进入了恢复模式。

不管是崩溃还是初次启动集群，选Leader的流程都是类似的：

- 每个服务器都会投票给自己以（myid（每个服务器的唯一id），zxid）的信息形式广播出去。
- 接受来自各个服务器的投票。集群的每个服务器收到投票后，首先判断该投票的有效性，如检查是否是本轮投票、是否来自 LOOKING 状态的服务器。
- 处理投票。针对每一个投票，服务器都需要将别人的投票和自己的投票进行比较：优先检查 ZXID。ZXID 比较大的服务器优先作为 Leader，如果 ZXID 相同，那么就比较 myid。myid 较大的服务器作为 Leader 服务器
- 统计投票。每次投票后，服务器都会统计投票信息，判断是否已经有过半机器接受到相同的投票信息，如果超过半数则终止投票。
- 一旦确定了 Leader，每个服务器就会更新自己的状态，如果是Follower，那么就变更为 FOLLOWING，果是 Leader，就变更为 LEADING。当同步完成，集群就可以正常的处理请求，进入消息广播模式。

#### 消息广播模式

- Leader 服务器接收到请求后在进行广播事务 Proposal 之前会为这个事务分配一个 ZXID，再进行广播
- Leader 服务器会为每个 Follower 服务器都各自分配一个单独的队列，然后将需要广播的事务 Proposal 依次放入这些队列中去，并根据 FIFO 策略进行消息的发送。
- 每个Follower 服务器在接收到后都会将其以事务日志的形式写入到本地磁盘中，并且在成功写入后返回 Leader 服务器一个 ACK 响应。
- 当有超过半数的服务器 ACK 响应后，Leader 就会广播一个 Commit 消息给所有的 Follower 服务器，Follower 接收到后就完成对事务的提交操作。

![消息广播模式](https://images.gitee.com/uploads/images/2021/0407/152947_5cdc9968_8076629.png "消息广播模式.png")

#### 同步模式

当旧的Leader崩溃后，新选出的Leader要保证以下两个问题：

- 需要确保那些已经在旧Leader服务器上提交的事务，最终被所有服务器都提交。

- 需要确保丢弃那些只在旧Leader服务器上被提交的事务。

每当选举产生一个新的Leader服务器，就会从这个Leader服务器上取出本地日志中最大事务propose的ZXID，然后解析出epoch，最后对epoch加1；低32位就从0开始重新生成新的ZXID。ZAB协议通过epoch编号来区分Leader周期变化的策略，来保证丢弃那些只在上一个Leader服务器上被提交的事务

同步主要包括两种：

- leader选举成功后初始化广播新的epoch，其他服务器接收到后可以从leader同步数据。

- leader提案通过后更新数据，然后发送更新广播，其他服务器接收到后更新该数据


# 分布式锁

分布式锁，这个主要得益于ZooKeeper为我们保证了数据的强一致性。锁服务可以分为两类，一个是保持独占，另一个是控制时序。
所谓保持独占，就是所有试图来获取这个锁的客户端，最终只有一个可以成功获得这把锁。通常的做法是把zk上的一个znode看作是一把锁，通过create znode的方式来实现。所有客户端都去创建 /distribute_lock 节点，最终成功创建的那个客户端也即拥有了这把锁。
控制时序，就是所有视图来获取这个锁的客户端，最终都是会被安排执行，只是有个全局时序了。做法和上面基本类似，只是这里 /distribute_lock 已绊预先存在，客户端在它下面创建临时有序节点（这个可以通过节点的属性控制：CreateMode.EPHEMERAL_SEQUENTIAL来指定）。Zk的父节点（/distribute_lock）维持一份sequence,保证子节点创建的时序性，从而也形成了每个客户端的全局时序


# 集群


参考:

[Zookeeper 数据结构详解](https://juejin.cn/post/6844904167287291912)

[分布式系列文章——Paxos算法原理与推导](https://www.cnblogs.com/linbingdong/p/6253479.html)

[Zookeeper协议篇-Paxos算法与ZAB协议](https://zhuanlan.zhihu.com/p/145305409)

[维基百科 Paxos算法](https://zh.wikipedia.org/wiki/Paxos%E7%AE%97%E6%B3%95)

[Paxos算法详解](https://zhuanlan.zhihu.com/p/31780743)

[ZAB协议和Paxos算法](https://my.oschina.net/OutOfMemory/blog/812947)

[实例详解ZooKeeper ZAB协议、分布式锁与领导选举](https://dbaplus.cn/news-141-1875-1.html)

[ZooKeeper理论基础（一）简介、Paxos 算法、ZAB协议](https://blog.csdn.net/weixin_41947378/article/details/106948374)