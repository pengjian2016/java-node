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

AOF（Append Only File ）持久化功能，它会把被执行的写命令(读命令不会记录)写到AOF文件的末尾，记录数据的变化。默认情况下，Redis是没有开启AOF持久化的，开启后，每执行一条更改Redis数据的命令，都会把该命令追加到AOF文件中，这是会降低Redis的性能，但大部分情况下这个影响是能够接受的，另外使用较快的硬盘可以提高AOF的性能。

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
- 比RDB更可靠，默认策略是每秒保存一次，这意味着最多只会丢失1秒钟内的数据
- AOF文件是一个只进行追加的日志文件，且写入操作是以Redis协议的格式保存的，内容是可读的，适合误删紧急恢复

AOF 缺点：
- 在相同的数据量下，AOF文件一般要比RDB文件更大，它是一个类似于日志记录的文件
- AOF的不同策略会影响持久化的速度，通常配置的每秒1次已经能获得比较高的性能，但可能还是会比RDB慢。

4.0 之后redis开始支持RDB和AOF的混合持久化，一般线上系统也是两种持久方式都开启，使用RDB快速恢复数据，然后再通过AOF恢复5分钟内丢失的数据等。

### RDB 实现原理？

redis 定时任务函数serverCron ，该方法执行了rdb的定时任务，由于代码过长没有贴处理，有兴趣的可以看一下源码，其中最主要的是调用了rdbSaveBackground方法，rdbSaveBackground函数中最主要的工作就是调用 fork 命令生成子流程，然后在子流程中执行 rdbSave函数，这个是实际进行RDB持久化的方法，它的主要功能：
- 先创建一个临时文件用于保存数据
- 遍历redis数据库，将内存中的数据写入临时文件
- 临时文件重命名为正式RDB文件，覆盖原文件
- 更新持久化状态信息（dirty、lastsave）等

![输入图片说明](https://images.gitee.com/uploads/images/2021/0426/105732_d072365b_8076629.png "rdbSave过程.png")

```
// https://github.com/redis/redis/blob/6.2/src/server.c
/* This is our timer interrupt, called server.hz times per second.
 * Here is where we do a number of things that need to be done asynchronously.
 * For instance:
 *
 * - Active expired keys collection (it is also performed in a lazy way on
 *   lookup).
 * - Software watchdog.
 * - Update some statistic.
 * - Incremental rehashing of the DBs hash tables.
 * - Triggering BGSAVE / AOF rewrite, and handling of terminated children.
 * - Clients timeout of different kinds.
 * - Replication reconnection.
 * - Many more...
 *
 * Everything directly called here will be called server.hz times per second,
 * so in order to throttle execution of things we want to do less frequently
 * a macro is used: run_with_period(milliseconds) { .... }
 */
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
    ...
    rdbSaveBackground()
}
// 调用fork函数
int redisFork(int purpose) {
    ...
}
// https://github.com/redis/redis/blob/6.2/src/rdb.c
int rdbSaveBackground(char *filename, rdbSaveInfo *rsi) {antirez, 4 years ago: • PSYNC2: different improvements to Redis repli…
    pid_t childpid;

    if (hasActiveChildProcess()) return C_ERR;

    server.dirty_before_bgsave = server.dirty;
    server.lastbgsave_try = time(NULL);

    if ((childpid = redisFork(CHILD_TYPE_RDB)) == 0) {
        int retval;

        /* Child */
        redisSetProcTitle("redis-rdb-bgsave");
        redisSetCpuAffinity(server.bgsave_cpulist);
        retval = rdbSave(filename,rsi);
        if (retval == C_OK) {
            sendChildCowInfo(CHILD_INFO_TYPE_RDB_COW_SIZE, "RDB");
        }
        exitFromChild((retval == C_OK) ? 0 : 1);
    } else {
        /* Parent */
        if (childpid == -1) {
            server.lastbgsave_status = C_ERR;
            serverLog(LL_WARNING,"Can't save in background: fork: %s",
                strerror(errno));
            return C_ERR;
        }
        serverLog(LL_NOTICE,"Background saving started by pid %ld",(long) childpid);
        server.rdb_save_time_start = time(NULL);
        server.rdb_child_type = RDB_CHILD_TYPE_DISK;
        return C_OK;
    }
    return C_OK; /* unreached */
}
/* Save the DB on disk. Return C_ERR on error, C_OK on success. */
int rdbSave(char *filename, rdbSaveInfo *rsi) {
}
```
### RDB 会将已经过期的KEY持久化吗？

如果我们不看代码，可能会猜测它不会将已经过期的key持久化，那么到底是不是这样呢？

rdbSave方法会调用rdbSaveRio，这里会遍历数据库中的所有key，获取每个key的过期时间（如果有的话），通过rdbSaveKeyValuePair方法保存，然而在这个方法里面只是保存过期时间，并不会与当前时间比较（有些文章贴出了if (expiretime < now) return 0; 这样的判断，然而那是4.0以前的版本了），所以过期的key 也会被RDB记录下来吗？为什么要记录下来呢？，带着这样的疑问我到社区提出了问题，不过目前还没有人回复，后续跟进。

```
// https://github.com/redis/redis/blob/6.2/src/rdb.c
int rdbSaveRio(rio *rdb, int *error, int rdbflags, rdbSaveInfo *rsi) {
    ...
     while((de = dictNext(di)) != NULL) {
            sds keystr = dictGetKey(de);
            robj key, *o = dictGetVal(de);
            long long expire;

            initStaticStringObject(key,keystr);
            expire = getExpire(db,&key);
            if (rdbSaveKeyValuePair(rdb,&key,o,expire) == -1) goto werr;
            ...
}
int rdbSaveKeyValuePair(rio *rdb, robj *key, robj *val, long long expiretime) {
    ...
    /* Save the expire time */
    if (expiretime != -1) {
        if (rdbSaveType(rdb,RDB_OPCODE_EXPIRETIME_MS) == -1) return -1;
        if (rdbSaveMillisecondTime(rdb,expiretime) == -1) return -1;
    }
    ...
}
```
### RDB 数据载入过程？
RDB 功能最核心的是 rdbSave 和 rdbLoad 两个函数，前者用于持久化，上面已经介绍过，后者用于将 RDB 文件中的数据重新载入到内存中。话不多说，贴部分源码：

```
//https://github.com/redis/redis/blob/6.2/src/rdb.c
int rdbLoad(char *filename, rdbSaveInfo *rsi, int rdbflags) {
    FILE *fp;
    rio rdb;
    int retval;

    if ((fp = fopen(filename,"r")) == NULL) return C_ERR;
    startLoadingFile(fp, filename,rdbflags);
    rioInitWithFile(&rdb,fp);
    retval = rdbLoadRio(&rdb,rdbflags,rsi);
    fclose(fp);
    stopLoading(retval==C_OK);
    return retval;
}
int rdbLoadRio(rio *rdb, int rdbflags, rdbSaveInfo *rsi) {
    ...
    while(1) {
        sds key;
        robj *val;
        ...

        /* Check if the key already expired. This function is used when loading
         * an RDB file from disk, either at startup, or when an RDB was
         * received from the master. In the latter case, the master is
         * responsible for key expiry. If we would expire keys here, the
         * snapshot taken by the master may not be reflected on the slave.
         * Similarly if the RDB is the preamble of an AOF file, we want to
         * load all the keys as they are, since the log of operations later
         * assume to work in an exact keyspace state. */
        if (iAmMaster() &&
            !(rdbflags&RDBFLAGS_AOF_PREAMBLE) &&
            expiretime != -1 && expiretime < now)
        {
            sdsfree(key);
            decrRefCount(val);
        } else {
            robj keyobj;
            initStaticStringObject(keyobj,key);

            /* Add the new object in the hash table */
            int added = dbAddRDBLoad(db,key,val);
            ...
}
```
由于代码过长，没办法全部贴出来，rdbLoad 主要过程就是读取RDB文件，将数据加载到redis数据库中，其间会对过期时间判断，如果已经过期了，会释放key。

### AOF 持久化过程？

AOF 大致的流程可以分为两步：
- 命令实时写入时，增量追加到文件，主要流程为：命令写入-> 保存到AOF 缓冲区-> 同步到磁盘AOF文件中
- AOF 文件重写，AOF文件过大时触发重写操作（触发规则可以在配置文件中配置），流程：redis fork一个子进程->子进程基于当前内存中的数据，构建日志，开始往一个新的临时的AOF文件中写入日志->redis主进程，接收到client新的写操作之后，在内存中写入日志，同时新的日志也继续写入旧的AOF文件->子进程写完新的日志文件之后，redis主进程将内存中的新日志再次追加到新的AOF文件中->用新的日志文件替换掉旧的日志文件

### AOF 中过期的key如何处理？

AOF 是已文件追加的方式记录写操作，那么当redis中的key过期了，发生惰性删除或定期删除key时，AOF文件会记录这个过程吗？

当过期键被惰性删除或者定期删除之后，程序会向AOF文件追加（append）一条DEL命令，来显式地记录该键已被删除。

在执行AOF重写的过程中，程序会对数据库中的键进行检查，已过期的键不会被保存到重写后的AOF文件中。

### 数据恢复优先级？

如果 Redis 同时使用 RDB 和 AOF 持久化，Redis 会优先使用 AOF 进行恢复数据

# redis 命令执行过程？


参考

[Redis 多线程网络模型全面揭秘](https://strikefreedom.top/multiple-threaded-network-model-in-redis)

[Go netpoller 原生网络模型之源码全面揭秘](https://strikefreedom.top/go-netpoll-io-multiplexing-reactor)

[Unix网络编程的5种I/O模型](https://zhuanlan.zhihu.com/p/121826927)

[文件事件](http://redisbook.com/preview/event/file_event.html)

[Redis持久化机制：RDB和AOF](https://juejin.cn/post/6844903939339452430)

[Redis专题：持久化方式之RDB](https://segmentfault.com/a/1190000039208707)