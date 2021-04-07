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

Paxos算法是基于消息传递且具有高度容错特性的共识（consensus）算法，是目前公认的解决分布式一致性问题最有效的算法之一。Zookeeper 采用paxos算法保证了数据的一致性。

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


# Multi-Paxos算法(zookeeper 使用一个类Multi-Paxos的共识算法作为底层存储协同的机制)

原始的Paxos算法（Basic Paxos）只能对一个值形成决议，决议的形成至少需要两次网络来回，在高并发情况下可能需要更多的网络来回，极端情况下甚至可能形成活锁。如果想连续确定多个值，Basic Paxos搞不定了。因此Basic Paxos几乎只是用来做理论研究，并不直接应用在实际工程中。

实际应用中几乎都需要连续确定多个值，而且希望能有更高的效率。Multi-Paxos正是为解决此问题而提出。Multi-Paxos基于Basic Paxos做了两点改进：

- 针对每一个要确定的值，运行一次Paxos算法实例（Instance），形成决议。每一个Paxos实例使用唯一的Instance ID标识。
- 在所有Proposers中选举一个Leader，由Leader唯一地提交Proposal给Acceptors进行表决。这样没有Proposer竞争，解决了活锁问题。在系统中仅有一个Leader进行Value提交的情况下，Prepare阶段就可以跳过，从而将两阶段变为一阶段，提高效率。

Multi-Paxos首先需要选举Leader，Leader的确定也是一次决议的形成，所以可执行一次Basic Paxos实例来选举出一个Leader。选出Leader之后只能由Leader提交Proposal，在Leader宕机之后服务临时不可用，需要重新选举Leader继续服务。在系统中仅有一个Leader进行Proposal提交的情况下，Prepare阶段可以跳过。

Multi-Paxos通过改变Prepare阶段的作用范围至后面Leader提交的所有实例，从而使得Leader的连续提交只需要执行一次Prepare阶段，后续只需要执行Accept阶段，将两阶段变为一阶段，提高了效率。为了区分连续提交的多个实例，每个实例使用一个Instance ID标识，Instance ID由Leader本地递增生成即可。



# Zab 协议

# 分布式锁

# 集群


参考:

[Zookeeper 数据结构详解](https://juejin.cn/post/6844904167287291912)

[分布式系列文章——Paxos算法原理与推导](https://www.cnblogs.com/linbingdong/p/6253479.html)

[Zookeeper协议篇-Paxos算法与ZAB协议](https://zhuanlan.zhihu.com/p/145305409)

[维基百科 Paxos算法](https://zh.wikipedia.org/wiki/Paxos%E7%AE%97%E6%B3%95)

[Paxos算法详解](https://zhuanlan.zhihu.com/p/31780743)