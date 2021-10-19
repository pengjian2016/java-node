# redis 分布式锁，锁的续期问题等

分布式锁有基于数据库的、zookeeper和redis，数据库实现的分布式锁是传统的加锁方式，基于某条记录用悲观锁或乐观锁等，zookeeper如何实现分布式锁已经在 [zookeeper 数据结构、Paxos算法，Zab协议，分布式锁，集群](https://gitee.com/javajov/java-senior-engineer-interview/blob/master/mq/zk.md) 这一节中已经介绍过
 
现在主要介绍redis 实现分布式锁：

Redis 锁主要利用 Redis 的 SETNX key value 命令，即 "SET if Not eXists" 如果key不存在，则设置成功并返回1，如果存在，则返回0

不过这样设置解锁的时候必须执行删除命令，其他线程才能拿到锁，如果服务器down了，或者后续执行删除的命令没执行，那么这个锁将永远无法拿到，因此我们需要给锁设置一个过期时间，EXPIRE Key Seconds，但是设置锁和设置过期时间是两个命令，无法保证原子性操作，比如设置过期失败了，这样又导致之前的问题。好在redis 提供了set 扩展命令(也会有使用setnx+lua脚本的方式)：

```
# set key value ex time nx 成功返回ok，失败返回空
set mylock 1 ex 10000 nx
ok
```
![Image description](https://images.gitee.com/uploads/images/2021/1015/102350_e3f1149a_8076629.png "屏幕截图.png")

这样redis 加锁有了，锁也有了过期时间，即使出现异常也能避免死锁，但是另外的问题也来了：

##### 1. 假如业务过长，key过期了，新的线程获取到了锁并设置成功，但是之前的线程又继续执行了 然后删除了 key，这样新的线程等于加锁失败了。

##### 2. 过期时间是固定的，如果过期时间内，业务仍然没有执行完成怎么办呢？

##### 3. redis 集群情况下，假设A线程在master节点上已经获取到了锁，但是锁的信息并未同步到slave节点，然后刚好 master节点down机，从节点升级为主节点，B线程从新的主节点中获取到了锁，这样A线程已经拿到了锁，B线程也有了锁，这就造成了多个客户端都拿到锁的情况

##### 问题一：设置key的value时，可以使用如UUID这样的随机串，或者当前线程的id等，在删除之前先判断key对于的value值是否是当前线程设置的值，是才进行删除，否则不执行；

##### 问题二：加锁时先设置一个过期时间，然后开启一个「守护线程」，定时去检测这个锁的失效时间，如果锁快要过期了，操作共享资源还未完成，那么就自动对锁进行「续期」，重新设置过期时间，这个过程比较复杂，好在这些东西 在Redisson中已经封装好了，关于Redisson下面会介绍。

##### 问题三：redlock算法，在Redis的分布式环境中，我们假设有N个Redis master。这些节点完全互相独立，不存在主从复制或者其他集群协调机制。我们确保将在N个实例上使用与在Redis单实例下相同方法获取和释放锁。现在我们假设有5个Redis master节点，同时我们需要在5台服务器上面运行这些Redis实例，这样保证他们不会同时都宕掉，为了取到锁，客户端应该执行以下操作:

- 获取当前Unix时间，以毫秒为单位。

- 依次尝试从5个实例，使用相同的key和具有唯一性的value（例如UUID）获取锁。当向Redis请求获取锁时，客户端应该设置一个网络连接和响应超时时间，这个超时时间应该小于锁的失效时间。例如你的锁自动失效时间为10秒，则超时时间应该在5-50毫秒之间。这样可以避免服务器端Redis已经挂掉的情况下，客户端还在死死地等待响应结果。如果服务器端没有在规定时间内响应，客户端应该尽快尝试去另外一个Redis实例请求获取锁。

- 客户端使用当前时间减去开始获取锁时间（步骤1记录的时间）就得到获取锁使用的时间。当且仅当从大多数（N/2+1，这里是3个节点）的Redis节点都取到锁，并且使用的时间小于锁失效时间时，锁才算获取成功。

- 如果取到了锁，key的真正有效时间等于有效时间减去获取锁所使用的时间（步骤3计算的结果）。

- 如果因为某些原因，获取锁失败（没有在至少N/2+1个Redis实例取到锁或者取锁时间已经超过了有效时间），客户端应该在所有的Redis实例上进行解锁（即便某些Redis实例根本就没有加锁成功，防止某些节点获取到锁但是客户端没有得到响应而导致接下来的一段时间不能被重新获取锁）。

这段描述也非常的复杂，好在redisson中也已经实现了redlock的算法，另外还有两位大佬关于 redlock算法的激烈讨论，参考这篇文章：http://kaito-kidd.com/2021/06/08/is-redis-distributed-lock-really-safe/

#### Redisson
这是一个java封装的redis相关的操作命令（类似jedis,比它实现了更多东西），开源地址：https://github.com/redisson/redisson

简单看一下官方给的示例：https://github.com/redisson/redisson/wiki/8.-distributed-locks-and-synchronizers/#81-lock

```
RLock lock = redisson.getLock("myLock");

// 加锁方法1，拿锁失败时会不停的重试，具有Watch Dog 自动延期机制 默认续30s 每隔10秒，续到30s
lock.lock();

// 加锁方法2，获取锁并在10秒后自动解锁，没有Watch Dog ，10s后自动释放
lock.lock(10, TimeUnit.SECONDS);

// 加锁方法3 等待锁定时间长达100秒后，10秒后自动释放锁，没有Watch Dog 
boolean res = lock.tryLock(100, 10, TimeUnit.SECONDS);
if (res) {
   try {
     ...
   } finally {
       lock.unlock();
   }
}

```
Redisson提供了一个监控锁的看门狗，它的作用是在Redisson实例被关闭前，不断的延长锁的有效期，也就是说，如果一个拿到锁的线程一直没有完成逻辑，那么看门狗会帮助线程不断的延长锁超时时间，锁不会因为超时而被释放。

RLock 与 ReentrantLock 类似，也实现了java 的lock接口，所以它也有公平锁非公平锁之分，另外也有读写锁等，这里就不再过多的展开了，有兴趣的可以看一下官方的文档。


参考:

[Redlock：Redis分布式锁最牛逼的实现](https://mp.weixin.qq.com/s?__biz=MzU5ODUwNzY1Nw==&mid=2247484155&idx=1&sn=0c73f45f2f641ba0bf4399f57170ac9b&scene=21#wechat_redirect)

强烈推荐：[深度剖析：Redis分布式锁到底安全吗？看完这篇文章彻底懂了！](http://kaito-kidd.com/2021/06/08/is-redis-distributed-lock-really-safe/)

[Redisson 分布式锁实战与 watch dog 机制解读](https://www.cnblogs.com/keeya/p/14332131.html)


### 各分布式锁对比

|  | 数据库 | redis | zookeeper  |
|-----|-------|-----------|---|
|  优点 |  直接使用数据库，实现方式简单，不需要额外增加其他中间件（大多数系统都有数据库）    |   redis读写速度快，效率非常高        | 不存在redis的超时、数据同步问题、主从切换（zookeeper主从切换的过程中服务是不可用的）的问题，可靠性很高  |
|  缺点 |  数据库操作性能较差；而且要实现可靠的分布式锁，它也有redis实现分布式锁的那些问题，比如锁的超时时间，数据库单节点挂掉如何解决等，复杂程度不比其他中间件低   |     依赖中间件，如果没有像redisson这种客户端工具，实现可靠的分布式锁比较复杂      | 保证了可靠性的同时牺牲了一部分效率（但是依然很高），性能不如redis（zookeeper要频繁的创建节点和删除节点） |

从实现的复杂性角度（从低到高）: Zookeeper >= redis > 数据库

从性能角度（从高到低）: redis > Zookeeper >= 数据库

从可靠性角度（从高到低）Zookeeper > redis > 数据库