# redis 为什么这么快？

redis 4.0 之前是单线程的，4.0 之后引入了多线程处理异步任务，这个时候它还只是针对哪些操作比较耗时的命令开启异步化，避免线程阻塞，6.0的时候redis采用了真正的I/O多线程模型 来处理网络请求。

说实话其实我也不知道该如何回答这个问题，在此之前我在面试的时候一般这么回答：

- 1. 单线程，省去了多线程上下文切换的开销，没有共享资源加锁的开销
- 2. 纯内存操作，同时高效的数据结构和编码方式，都是它更加快的原因之一
- 3. 使用IO多路复用技术，保证在监听多个Socket连接的情况下，只针对有活动的（有真正读写事件的连接）Socket采取反应

这里面2和3两条，面试的时候仍然适用，只是单线程这一块，就有待商榷了。

6.0 之前的版本，Redis 的核心网络模型仍然是单线程的，在6.0 之后，正式在核心网络模型中引入了多线程，也就是所谓的 I/O threading，这个时候才真正认为redis是多线程的。

那么为什么不继续使用单线程了？单线程的瓶颈在哪里？多线程的好处是什么？在回答这些问题之前，需要先了解一下什么是I/O多路复用。

# 什么是I/O多路复用？

> 所谓 I/O 多路复用指的就是 select/poll/epoll 这一系列的多路选择器：支持单一线程同时监听多个文件描述符（I/O 事件），阻塞等待，并在其中某个文件描述符可读写时收到通知。 I/O 复用其实复用的不是 I/O 连接，而是复用线程，让一个 thread of control 能够处理多个连接（I/O 事件）[摘自Go netpoller 原生网络模型之源码全面揭秘](https://strikefreedom.top/go-netpoll-io-multiplexing-reactor)

> select，poll以及大名鼎鼎的epoll就是IO多路复用模型，其特点就在于单个系统调用可以同时处理多个网络连接的IO，它的基本原理就是select/poll/epoll这个function会不断的轮询所负责的所有socket，当某个socket有数据到达了，就通知用户进程。当用户进程调用了select/poll/epoll，整个进程会被阻塞，而同时，kernel会“监视”所有select/poll/epoll负责的socket，当任何一个socket中的数据准备好了，select/poll/epoll就会返回。这个时候用户进程再调用recvfrom操作，将数据从内核缓冲区拷贝到用户进程缓冲区。[摘自Unix网络编程的5种I/O模型](https://zhuanlan.zhihu.com/p/121826927)

虽然定义略有差异，但是内容都差不多。

总结IO多路复用：一个线程利用select/poll/epoll 这些函数，能够监听多个连接（socket），当这些连接真正有读写等事件时，才会进行处理，否则就阻塞在那里。

回到前面的问题，redis 6.0 之前，它的核心网络模型一直是一个典型的 单Reactor 模型（可以理解为设计模式，而IO多路复用为该模式的一种实现），利用多路复用技术，在单线程的事件循环中不断去处理客户端请求，最后回写响应数据到客户端。

![输入图片说明](https://images.gitee.com/uploads/images/2021/0423/102239_89e00de5_8076629.png "屏幕截图.png")

6.0 之后引入多线程之后会进化为 Multi-Reactors 模式，这种模式不再是单线程的事件循环，而是有多个线程（Sub Reactors）各自维护一个独立的事件循环，由 Main Reactor 负责接收新连接并分发给 Sub Reactors 去独立处理，最后 Sub Reactors 回写响应给客户端。

![输入图片说明](https://images.gitee.com/uploads/images/2021/0423/102249_a1e0b598_8076629.png "屏幕截图.png")

那么为什么从单线程变成了多线程呢？

redis官方认为，因为Redis是基于内存的操作，CPU不是Redis的瓶颈，Redis的瓶颈最有可能是机器 **内存** 或者 **网络** 带宽，使用单线程后，可维护性高。多线程模型虽然在某些方面表现优异，但是它却引入了程序执行顺序的不确定性，带来了并发读写的一系列问题，增加了系统复杂度、同时可能存在线程切换、甚至加锁解锁、死锁造成的性能损耗。

单线程瓶颈，并发量非常大时，单线程读写客户端IO数据存在性能瓶颈，读写客户端数据依旧是同步IO，只能单线程依次读取客户端的数据，无法利用到CPU多核。为了提升QPS，很多公司的做法是部署Redis集群，并且尽可能提升Redis机器数，但是这种做法的资源消耗是巨大的。互联网业务越来越复杂，有些公司动不动就上亿的交易量，因此需要更大的QPS。为了提升网络IO的性能，redis因而引入了多线程。需要注意的是，Redis 6.0 只有在网络请求的接收和解析，以及请求后的数据通过网络返回给时，使用了多线程，命令的执行还是单线程的。


# 什么是事件派发器？（或者文件事件派发器）

说到redis的IO多路复用，可能大家都见过下面这张图：

![输入图片说明](https://images.gitee.com/uploads/images/2021/0425/093117_18cca787_8076629.png "屏幕截图.png")

I/O多路复用程序负责监听多个套接字，并向文件事件派发器传递那些产生了事件的套接字（socket）。

文件事件是对套接字操作的抽象，每当一个套接字准备好执行 accept、read、write和 close 等操作时，就会产生一个文件事件。因为 Redis 通常会连接多个套接字，所以多个文件事件有可能并发的出现,但I/O多路复用程序总是会将所有产生的套接字都放到同一个队列里，然后文件事件处理器会以有序、同步、单个套接字的方式处理该队列中的套接字，也就是处理就绪的文件事件。

# RDB和AOF 持久化

RDB 持久化快照，在一定间隔时间内将内存中的数据据以快照的形式保存到磁盘中。

在redis.conf 配置文件中，RDB 默认是开启的，持久化后的文件名称默认为dump.rdb：

```
# Save the DB on disk:
#
#   save <seconds> <changes>
#
#   Will save the DB if both the given number of seconds and the given
#   number of write operations against the DB occurred.
#
#   In the example below the behavior will be to save:
#   after 900 sec (15 min) if at least 1 key changed
#   after 300 sec (5 min) if at least 10 keys changed
#   after 60 sec if at least 10000 keys changed
#
#   Note: you can disable saving completely by commenting out all "save" lines.
#
#   It is also possible to remove all the previously configured save
#   points by adding a save directive with a single empty string argument
#   like in the following example:
#  
#   save ""   // 如果要关闭 RDB 则使用这个命令

save 900 1     // 每15分钟，至少发生1次key的变动，会触发持久化 
save 300 10    // 每5分钟，至少发生10次key的变动，会触发持久化 
save 60 10000  // 每分钟，至少发生10000次key的变动，会触发持久化  

# 持久化文件名
dbfilename dump.rdb
```
以上是通过配置文件，间隔时间内自动触发的方式，当然我们也可以手动使用命令触发RDB持久化：

- save 命令，该命令是同步操作，会阻塞当前Redis服务器，执行save命令期间，Redis不能处理其他命令，直到RDB过程完成为止。（线上基本不会使用该命令）
- bgsave 命令，异步操作，Redis fork 出一个新子进程，原来的 Redis 进程（父进程）继续处理客户端请求，而子进程则负责将数据保存到磁盘，然后退出。（上面配置文件中，也是使用的bgsave命令）

AOF（Append Only File ）持久化功能，它会把被执行的写命令写到AOF文件的末尾，记录数据的变化。默认情况下，Redis是没有开启AOF持久化的，开启后，每执行一条更改Redis数据的命令，都会把该命令追加到AOF文件中，这是会降低Redis的性能，但大部分情况下这个影响是能够接受的，另外使用较快的硬盘可以提高AOF的性能。

如何开启AOF，在redis.conf 配置文件中：

```

# 开启AOF 持久化
appendonly yes

# aof持久化文件名称
appendfilename "appendonly.aof"

# 同步策略，always-每次有写命令，都会同步写入到磁盘，everysec-每秒执行一次同步，no-由操作系统来决定何时同步
# appendfsync always
appendfsync everysec
# appendfsync no

```
RDB和AOF 各自的优缺点


RDB 优点：
- 1. RDB快照是一个压缩过的非常紧凑的文件，保存着某个时间点的数据集，适合做数据的备份，灾难恢复
- 2. RDB是fork子进程进行持久化，对父进程影响不大，这保证了 redis 的高性能
- 3. 数据恢复时，数据较大的情况下，RDB要比AOF恢复更快

RDB 缺点：
- rdb是间隔一定时间执行，相比AOF来说，可能会丢失更多的数据
- 当redis中的数据较多时，fork的子进程也会消耗更多的CPU资源，可能会导致redis服务卡顿的现象

AOF 优点：
- 比RDB更可靠，默认策略时每秒保存一次，这样最多只会丢失1秒钟内的数据
- AOF文件是一个只进行追加的日志文件，且写入操作是以Redis协议的格式保存的，内容是可读的，适合误删紧急恢复

AOF 缺点：
- 在相同的数据量下，AOF文件一般要比RDB文件更大，它是一个类似于日志记录的文件
- AOF的不同策略会影响持久化的速度，通常配置的每秒1次已经能获得比较高的性能，可能还是会比RDB慢。


不过 4.0 之后redis开始支持RDB和AOF的混合持久化，一般线上系统也是两种持久方式都开启，使用RDB快速恢复数据，然后再通过AOF恢复5分钟内丢失的数据等。

数据如何恢复？

- 如果是redis进程挂掉，那么重启redis进程即可，直接基于AOF文件恢复数据

### RDB 如何fork子进程？

### RDB 会将已经过期的数据持久化吗？

### AOF 持久化过程？


# 数据库和缓存一致性问题是如何解决？

# redis 命令执行过程？



参考

[Redis 多线程网络模型全面揭秘](https://strikefreedom.top/multiple-threaded-network-model-in-redis)

[Go netpoller 原生网络模型之源码全面揭秘](https://strikefreedom.top/go-netpoll-io-multiplexing-reactor)

[Unix网络编程的5种I/O模型](https://zhuanlan.zhihu.com/p/121826927)

[文件事件](http://redisbook.com/preview/event/file_event.html)

[Redis持久化机制：RDB和AOF](https://juejin.cn/post/6844903939339452430)