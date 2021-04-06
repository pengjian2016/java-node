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

Paxos算法是基于消息传递且具有高度容错特性的一致性算法，是目前公认的解决分布式一致性问题最有效的算法之一

# Zab 协议


参考:

[Zookeeper 数据结构详解](https://juejin.cn/post/6844904167287291912)

[分布式系列文章——Paxos算法原理与推导](https://www.cnblogs.com/linbingdong/p/6253479.html)