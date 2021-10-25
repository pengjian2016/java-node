
1.ReentrantLock lock的过程
2.ElasticSearch 数据结构，深度检索过程
3.redis 跳表时间复杂度
4.volatile 
5.AQS 有哪些数据结构
6.ClassLoad，Class.forName()区别
7.JMM 内存模型
8.CMS，G1 哪个阶段会产生浮动垃圾，区别
CMS：初始标记，并发标记，重新标记，并发清除，在并发标记阶段会产生浮动垃圾，在初始标记和重新标记的过程会Stop the world  缺点：浮动垃圾，标记清除算法
G1:初始标记，并发标记，最终标记，筛选回收，在并发标记阶段会产生浮动垃圾，在初始标记和最终标记的过程会Stop the world
9.HashMap 如何计算数组长度,n为什么是2的次方
10.new 的过程 内存 分配在哪个区
12.spring  feign client 如何配置线程池
13. cloud，dubbo 研究一下

repeateable read 如何避免脏读的
为什么使用b+树，优缺点
bean的注入过程循环依赖问题，如何打破


Elasticsearch
它有master节点和数据节点，master也有主次之分
设置为master的节点启动时，去发现寻找其他有资格成为Master的节点，当发现的节点超过半数N/2+1时
可以被选举为Master，其他节点就加入该集群中。

1.写入过程
发送写请求的时候，先通过协调节点计算文档ID的hash值，找到要写入的分片节点，分片节点收到写请求后，先写入内存中（同时写入translog），然后定时1秒刷到文件缓存中，写到文件缓存之后这个文档可以被检索到，之后才会被写到磁盘中（默认30分钟或者日志过大的时候）。每个写请求都会记录translog日志，记录被写到磁盘之后会删除translog。

2.删除或更新
因为ES文档是不可更改的，删除的时候记录先被标记为del，检索的时候被过滤掉。更新的时候也是将旧记录标记为删除，插入新的记录。之后merge的时候会删除这些记录

3.搜索过程
协调节点通知所有的分片节点检索所有符合条件的记录（只带记录id和排序条件），汇总给协调节点，协调节点排序之后再去取完整的数据

4.并发读写一致性问题
可以通过版本号控制，新数据不会被旧数据覆盖调

5.Elasticsearch中的倒排索引
倒排序对文档内容进行分词，提出关键词，记录关键词的出现次数和位置，以及这个文档的ID，用户检索的时候找到倒排序中的关键词，根据关联的文档ID，取出文档记录。


kafka
broker就是服务，可以有多个topic，每个topic可有多个partion，不同的partion可以分布到不同的broker上，所以天然支持集群。kafka的消费者是pull模式，每个消费者携带offset，

集群，Leader是在zookeeper下创建临时节点controller，这个节点只能有一个客户端能创建成功，其他的会抛异常
Leader脱机之后，节点会被删除，其他节点可以再次创建。

1.消息丢失的情况
生产者，有几种模式，0-发送消息后不需要确认，1-收到Leader的确认消息之后认为成功，-1-表示收到Leader和Foller之后才认为成功
所以可以把模式设置为-1，发送失败的时候重新发送，或者是发送之前保存消息到redis或者数据库，收到消息确认之后删除消息，否在就重发
消费者，可以关闭自动提交，进行手动提交，或者是保存成功之后给生产者确认消息

2.消息重复的问题
消费者：把处理过的消息ID放到缓存中，遇到已经有的就丢弃，或者是每个消息都有一个唯一主键，直接保存的时候报主键冲突就丢弃

3.消息顺序问题
topic的partion，同一个partion下的消息是有顺序的，每个都有offset，如果需要顺序可以只设置一个partion或者，根据key的规则，把相同的key，发送到相同的partion下

4.zookeeper的作用
作为broker的注册中心，管理broker
用来注册管理Topic
做生产者和消费者的负载均衡

zookeeper
是一个协调服务，集群：分为leader和follower，观察者。一个Leader、多个Follower。更新请求顺序执行。
Leader选举过程，A节点，选自己，询问B节点，同意，C节点同意
B几点选自己，询问A，我已经超过半数不同意，询问C，A已经操过半数不同意
C也是如此。基本上就是谁先启动谁当头。
节点分为
持久几点
临时节点
持久顺序节点
临时顺序节点

1.分布式锁
每个客户端会在锁节点下创建临时顺序节点，这个节点有个id，在判断是否能否获取锁的时候，会判断这个id是不是排在第一位，第一位才能获取锁，否在失败。

2.注册中心
服务提供者名称和ip注册到zookeeper中，保存下来，实际上是在zookeeper中创建数据节点，相同节点下有多个服务列表。长连接心跳，定期检查服务是否正常，多次失败后，删除该服务
消费者从zookeeper中拉取需要的服务列表，列表有变化后，会通知给消费者

redis

1.为什么不用多线程
多线程有上下文切换的过程，这个也是比较消耗资源的，它本身是纯内存操作，采用IO的多路复用，实际并不比多线程慢。IO的多路复用，根据不同的socket，转发到不同的事件处理器中

2.redis性能为什么高，如何利用多核cpu机器
单线程，减少了多线程的上下文的开销，内存操作。IO多路复用。数据结构做了优化
多核CPU利用，可以部署多个redis实例，每个示例跟CPU内核绑定（taskset）

2.redis的缓存淘汰策略
会对设置了过期的Key，定期随机抽查，过期的数据会被删除。还有惰性删除，访问这个key的时候发现已经过期了就删除调。但是仍然解决不了问题。这个时候就会有淘汰策略，有几种策略。比如：
内存不足的时候写入报异常
淘汰最近最少使用的key
随机淘汰key
淘汰使用次数最少的key
从设置了过期的key中选取，淘汰即将过期的key，淘汰最近最少使用的key，随机淘汰等

3.redis有哪几种数据结构
string 可变的字符数组类似于java的StringBuilder这些
list 存放多个字符串，底层是类似与java的LinkedList，双向链表，有序可重复。可以用来放评论列表，时间轴等
set 类似于java中的HastSet，无序不重复的。可以取交集，并集，差集。可以用来取共同关注列表，点赞次数，收藏次数
hash 类似于java的HashMap，但是rehash过程不一样，是渐进式rehash，使用两个hashmap，查询的时候两个map都查询，写入的时候写入新的hashmap中，后续的定时任务将旧的key计算新的hash值放到新的map中，全部迁移完后，代替旧的map

zset 有序不重复的集合，使用跳表数据结构，类似于金字塔结构，value有个打分可以用来做排序的权重，可以用来做排行榜

4.redi持久化
RDB，定期生成内存快照副本，全量备份，适合做灾备，数据恢复比较快
缺点：每次都是一个副本，存储占用比较大，定时是一般是30秒一次，太快的也不太适合，会丢失的数据多一些
AOF：文件追加，每次写操作记录操作日志（每秒记录一次）。后期通过日志回放来恢复。但是恢复的速度较慢
最多丢失1秒中的数据

5.数据库的双写一致性
先更新数据库，然后删除缓存，删除失败的话，后续可以重试，重试多次仍然失败，说明服务器有问题

6.缓存穿透和缓存雪崩
缓存穿透：请求不存在的key，导致redis失效，全都查询了数据库，给数据库造成压力
方案：针对参数进行校验，校验合法的key才认为有效，或者把不存在的key缓存到redis中，设置过期时间等
或者可以使用布隆过滤器，将库中的key在布隆过滤器中插入一遍，请求的时候先判断布隆过滤器是否存在。
缓存雪崩：缓存不可用导致请求都落到了数据库，数据库瘫痪，导致整个应用不可用
方案：如果是缓存穿透的情况导致的，如上
其他情况 ，提高redis的可用性，使用集群等，对这些方法进行限流降级，数据预热，先请求一遍数据，加载到缓存中去
缓存击穿：原本热点的key，突然过期了或者失效了导致请求落到数据库，进而引发问题
解决：不同的key设置不同的时间，预估可能的过期时间，在这个基础上延长，从数据库请求key的时候加锁，请求成功后写入缓存。


7.分布式锁
SETNX 方法，只能有一个客户端能设置成功，并设置超时时间，如果超时了还没有执行成功，其他客户端可以判断版本号
8.其他
HyperLogLog：估算统计
布隆过滤器：key经过多个hash算法，数组中的对应位置变为1，如果key不存在，则一定不存在，key存在则可能存在
可以用来防止缓存穿透，如果key在它这里不存在则一定不存在，存在后在查询redis和数据库

9.redis 集群
主从模式：单住多从，主节点挂掉后只能读
哨兵模式：在主从基础上加了个哨兵，监控主节点挂掉后，从节点选一个作为主节点
集群模式：多主多从，每一个主节点至少对应一个从节点，主节点挂掉之后从节点切换

Geo：将二维的坐标数据转换为一维的数据，用它可以查找附近的人

hystrix/sentinel
资源隔离:
线程池:为每一种请求创建线程池，可以重复利用，优点是：互相隔离，某个服务线程池满了之后不会影响其他的服务
缺点是引入了线程池，增加了排队和上下文切换的时间
信号量：先拿到信号量才能发请求，开销较小，不支持超时，已经获得信号量的请求只能继续执行，其他未获得的则可以直接返回失败

熔断：当失败率达到一定阈值之后进行熔断，快速失败，之后会每个一段时间放入一些请求，如果请求成功了，如果多次都成功了，就关闭熔断

降级：限制对资源访问的请求数量

Netty 
是一款基于 NIO 的网络通信框架,高并发、传输快、封装好
可以用来做RPC通信工具，可以基于它实现HTTP服务器
1.核心组件
Channel：提供网络接口的组件，NIOServerSocketChannel
EventLoop：负责监听网络事件，并调用事件处理器，处理相关IO操作
ChannelFuture：监听异步返回结果
2.线程模型
Reactor 模式基于事件驱动，采用多路复用将事件分发给相应的 Handler 处理，非常适合处理海量 IO 的场景

dubbo
微服务框架，有服务提供方，使用方，注册中心，负载均衡容错管理，监控统计
主要基于RPC通信，默认使用 Netty通信框架

0.服务注册与发现
服务提供方启动后向注册中心注册自己的服务，使用方向注册中心订阅自己需要的服务
，提供方下线后，通知注册中心删除服务，使用方更新服务列表。

默认使用zookeeper作为注册中心，还可以使用redis作为注册中心
zookeeper保证cap定理的，CP，一致性和分区容错性

1.RPC协议
dubbo：单个长连接，NIO异步传输，传输数量小的场景，传输协议默认使用Netty，序列化：使用Hessian，还支持dubbo序列化协议
rmi：基于java rmi服务，序列化采用java 二进制序列化协议，多个短链接，BIO同步传输，适合常规RPC调用
hessian：基于Servlet容器的传输服务，使用hessian序列化协议，Http传输
http：传输

2.负载均衡策略
基于权重的随机
轮询
一致性hash算法：参数相同的路由到同一个服务器
最小活跃数等

3.容错策略
失败自动切换：失败几次之后，调用其他服务器
快速失败：只发起一次调用，失败立即报错
忽略失败：失败后，不抛异常给客户端
失败重试：失败后定时重试
并行调用：只要有一个成功，则返回
广播调用：只要有一个失败则报错

4.异步调用
Future 主动获取结果
Callback 异步回调
事件通知

5.Dubbo内置了哪几种服务容器
Spring container
Jetty Container
Log4j Container

通过 JDK 的 ShutdownHook  完成优雅停机

服务暴露 是在springbean实例化之后 调用 ServiceBean 的父类ServiceConfig方法中的export方法进行服务暴露

springcloud
基于http的 restful api 调用

注册中心 Eureka
保证 AP，可用性，分区容错性
自我保护机制
当Eureka Server 节点在短时间内丢失了过多实例的连接时（比如网络故障或频繁启动关闭客户端）节点会进入自我保护模式，保护注册信息，不再删除注册数据，故障恢复时，自动退出自我保护模式

负载均衡 Ribon

Feign：
整合了Ribon，具有负载的能力，又整合了hystix，有熔断限流的能力
基于动态代理机制，根据注解和选择的机器，拼接请求 url 地址，发起请求

熔断限流：hystrix
路由网关：zuul 动态路由
数据流：stream
消息总线：bus
可以用于广播配置文件的更

断路器：一段时间内失败次数过多或超时过多，则断路器打开，请求不会被发送到该服务，之后的话，会发送少量的请求，多次成功后关闭断路器
SpringCloudConfig：分布式配置中心组件，用来管理配置

spring
1.spring bean的生命周期
实例化，设置属性，setBeanName，setBeanFactory，setApplicationContext，预初始化方法，afterPropertiesSet（）方法，初始化方法，初始化之后的方法，destory方法
2.spring aop 
动态代理：被代理的对象实现了接口，使用JDK 动态代理（InvocationHandler），否则使用CGLIB
区别：CGLIb可以代理普通类，继承该类，调用方法的时候进行代理
主要：将业务上重复的功能抽取出来，统一处理，如：事务，创建链接，打开session，之后才进行业务，关闭session这些。

3.IOC
依赖注入，解耦，传统创建对象需要new一个，设置各种属性等，有了IOC之后，只需要在需要的地方使用注解，框架会帮助我们初始化对象。极大了减少了工作量。

4.bean 的注入方法
构造函数和属性注入
构造函数注入无法解决循环依赖问题

5.bean的作用域
singleton 单例，整个容器中只有一个
protoype 每次创建新对象
request 每个请求一个对象
session 整个session作用域内一个对象
global session 全局单例

6.springboot核心注解，EnableAutoConfiguration，CompantScan，Configuration

7.BeanFactory和ApplicationContext有什么区
 BeanFactory和ApplicationContext是Spring的两大核心接口，都可以当做Spring的容器。其中ApplicationContext是BeanFactory的子接口

BeanFactory：延迟加载来注入Bean，只有在使用到某个Bean时(调用getBean())，才对该Bean进行加载实例化
ApplicationContext：在容器启动时，一次性创建了所有的Bean（可以发现Spring中存在的配置错误）

8.Spring 中的单例 bean 的线程安全问题
单例 bean 存在线程问题：
解决：
在Bean对象中尽量避免定义可变的成员变量
在类中定义一个ThreadLocal成员变量

7. @Component 和 @Bean 的区别是什么
Component 作用于类，Bean作用于方法

8. 将一个类声明为Spring的 bean 的注解有哪些
Component ，Repository，Service，Controller

9. Spring中的设计模式
工厂设计模式：定义好产品接口，有不同的实现子类，有个工厂类，根据需要创建不同的子类对象
代理设计模式：在方法执行前后加入其他的逻辑
单例设计模式：全局唯一对象
模板方法模式：一个抽象类，定义好方法的执行顺序，调用抽象方法。子类实现抽象方法。
适配器模式：将一个接口适配成另一种接口

10.事务的隔离级别
默认级别，使用数据库的默认隔离级别
Read uncommit，读未提交
Read commit，避免脏读
Repeatable read，可以避免重复读
Serialable，最高级别，但是性能较差，加锁的条件多

11.事务的传播行为
如果当前有事务则加入，没有则创建事务
如果当前有事务则挂起，创建新事务执行
如果当前有事务则加入，没有则以非事务方式执行
如果当前有事务则加入，没有则抛异常
如果当前有事务则挂起，以非事务方式运行

12.springMVC 工作原理
请求发送到DispatchServlet，它找到HandlerMapping对应的Controler，处理完业务之后返回ModelView，view有ViewResolver找到对应的视图，DispatchServlet将model数据装入到View中，返回给浏览器加载

mysql
1.数据库引擎Innodb，支持事务，行级锁
2.索引
可以提供查询的速度，会影响插入更新删除的速度
索引占用存储空间，索引的类型：hash，B+
为什么查询快：
数据是有序的，平衡二叉树
聚集索引：以主键创建的索引，节点中存放的是整条记录
普通索引：节点中存放的是索引的列和主键，拿到主键后可能需要回表查询需要的数据，回表：通过主键查询数据，如果查询的数据是索引的列，则不需要回表

最左匹配原则：就是联合索引的时候，遇到范围查询或者只是部分列的时候，按照建索引的顺序匹配
覆盖索引：如果查询的字段刚好是索引的列，就可以直接返回值，不需要再回表查询

索引的创建
查询比较频繁的

SQL执行的过程
1.词法分析，提取关键词，查询条件，查询的表等
2.语法分析，确认SQL语句的正确性
3.对SQL进行优化，确定最好的执行方案
4.执行器，执行优化方案

mybatis