# 过期时间TTL说明

TTL 即 Time To Live，key的过期时间

设置方法(有兴趣的可以到 redis 在线测试工具尝试 https://try.redis.io/)
```
# set key value
set test 1
# expire key time 设置keyde过期时间以秒为单位，用pexpire key time 则是毫秒为单位
expire test 1000
# ttl key 显示key的剩余时间
ttl test
 993

```

下面是expire 命令的相关的部分代码:

```
源码:https://github.com/redis/redis/blob/6.2/src/expire.c
void expireGenericCommand(client *c, long long basetime, int unit) {
    robj *key = c->argv[1], *param = c->argv[2];
    long long when; /* unix time in milliseconds when the key will expire. */

    if (getLongLongFromObjectOrReply(c, param, &when, NULL) != C_OK)
        return;
    int negative_when = when < 0;
    if (unit == UNIT_SECONDS) when *= 1000;
    when += basetime;
    if (((when < 0) && !negative_when) || ((when-basetime > 0) && negative_when)) {
        /* EXPIRE allows negative numbers, but we can at least detect an
         * overflow by either unit conversion or basetime addition. */
        addReplyErrorFormat(c, "invalid expire time in %s", c->cmd->name);
        return;
    }
    /* No key, return zero. */
    if (lookupKeyWrite(c->db,key) == NULL) {
        addReply(c,shared.czero);
        return;
    }

    if (checkAlreadyExpired(when)) {
        robj *aux;

        int deleted = server.lazyfree_lazy_expire ? dbAsyncDelete(c->db,key) :
                                                    dbSyncDelete(c->db,key);
        serverAssertWithInfo(c,key,deleted);
        server.dirty++;

        /* Replicate/AOF this as an explicit DEL or UNLINK. */
        aux = server.lazyfree_lazy_expire ? shared.unlink : shared.del;
        rewriteClientCommandVector(c,2,aux,key);
        signalModifiedKey(c,c->db,key);
        notifyKeyspaceEvent(NOTIFY_GENERIC,"del",key,c->db->id);
        addReply(c, shared.cone);
        return;
    } else {
        setExpire(c,c->db,key,when);
        addReply(c,shared.cone);
        signalModifiedKey(c,c->db,key);
        notifyKeyspaceEvent(NOTIFY_GENERIC,"expire",key,c->db->id);
        server.dirty++;
        return;
    }
}
void expireCommand(client *c) {
    expireGenericCommand(c,mstime(),UNIT_SECONDS);
}

```
简单描述过程：

1. 获取参数中设置的过期时间 when；

2. 如果单位是 秒，则转换为毫秒级别，加上当前时间戳；

3. 如果不存在key，则返回，如果已经过期了，则删除；否则设置key的过期时间；

#### 既然key有过期时间的设置，那么key过期了是如何删除的呢？

##### 1.惰性删除

当 key 过期后，并不会立刻删除，即它们占用的内存不能够得到及时释放。而是当我们对这个key进行读或写的时候检查key是否有过期，如果过期了则删除key，这种方式被称为惰性删除。

#### 2.定期清理
如果只是惰性删除，很明显，如果这些key一直都不访问，那内存不是一直无法被释放吗？所以除此之外，它还有定时周期性的清理任务，当然这个定时清理的任务也只是随机抽取一定数量的key，检查有没有过期，如果过期了进行删除。


以上是关于ttl的简单说明，以及过期key的删除介绍。

如果长期将Redis作为缓存使用，内存难免会有使用完的时候，当Redis内存超出物理内存限制时，内存数据就会与磁盘产生频繁交换，使Redis性能急剧下降。此时如何淘汰数据释放空间呢？这就是下面要介绍的内存淘汰机制。

# 内存淘汰机制（LRU，LFU）


redis中的内存淘汰策略：

```
noeviction	默认策略，不淘汰key；大部分写命令都将返回错误（DEL等少数除外）
allkeys-random	从所有key中随机挑key据淘汰
volatile-random	从设置了过期时间的key中随机挑选数据淘汰
volatile-ttl	从设置了过期时间的key中，挑选即将过期的key进行删除
allkeys-lru	从所有key中根据 LRU 算法挑选数据淘汰，即从所有key中淘汰最近最少使用的key
volatile-lru	从设置了过期时间的key中根据 LRU 算法挑选数据淘汰，即从设置了过期的key中淘汰最近最少使用的key

allkeys-lfu	从所有key中根据 LFU 算法挑选数据淘汰（4.0及以上版本可用），即从所有key中淘汰最近访问次数最少的key
volatile-lfu	从设置了过期时间的key中根据 LFU 算法挑选数据淘汰（4.0及以上版本可用），即从设置了过期的key中淘汰最近访问次数最少的key
```
LRU 即最近最少使用，如果一个数据在最近一段时间没有被访问到，那么在将来它被访问的可能性也很小。所以，当指定的空间已存满数据时，应当把最久没有被访问到的数据淘汰。

LRU 算法的问题是，如果某些key 很长时间没有被访问，而在淘汰之前突然间访问了一次，结果因为最近被使用过而没有被淘汰，但实际可能后续也是很长实际不被访问。而某些key，一直都访问很频繁，只是在淘汰之前变少了，结果被淘汰了，后续可能仍然很频繁，这样这些key被删除后，导致请求落到了数据库上等，从而影响系统的稳定性等。

下面这张图简单描述了这种情况：

```
~~~~~A~~~~~A~~~~~A~~~~A~~~~~A~~~~~A~~|
~~B~~B~~B~~B~~B~~B~~B~~B~~B~~B~~B~~B~|
~~~~~~~~~~C~~~~~~~~~C~~~~~~~~~C~~~~~~|
~~~~~D~~~~~~~~~~D~~~~~~~~~D~~~~~~~~~D|
```
LRU淘汰算法可能将D保留了，而A和B这种访问频繁的反而被淘汰了。

鉴于这些情况，Redis4.0之后为引入了LFU算法，即淘汰最近使用次数最少的，它在原来LRU的基础上为每个key增加了一个计数器。

Redis 在实现 LFU 策略的时候，只是把原来 24bit 大小的 lru 字段，又进一步拆分成了两部分：

1. ldt 值：lru 字段的前 16bit，表示数据的访问时间戳；

2 .counter 值：lru 字段的后 8bit，表示数据的访问次数。

当 LFU 策略筛选数据时，Redis 会在候选集合中，根据数据 lru 字段的后 8bit 选择访问次数最少的数据进行淘汰。当访问次数相同时，再根据 lru 字段的前 16bit 值大小，选择访问时间最久远的数据进行淘汰。

counter 值 增加的规则是：每当数据被访问一次时，首先，用计数器当前的值乘以配置项 lfu_log_factor 再加 1，再取其倒数，得到一个 p 值；然后，把这个 p 值和一个取值范围在（0，1）间的随机数 r 值比大小，只有 p 值大于 r 值时，计数器才加 1。正是因为使用了非线性递增的计数器方法，即使缓存数据的访问次数成千上万，LFU 策略也可以有效地区分不同的访问次数，从而进行合理的数据筛选

当然有增加也会有衰减：LFU 策略会计算当前时间和数据最近一次访问时间的差值，并把这个差值换算成以分钟为单位。然后，LFU 策略再把这个差值除以 lfu_decay_time 值，所得的结果就是数据 counter 要衰减的值。假设 lfu_decay_time 取值为 1，如果数据在 N 分钟内没有被访问，那么它的访问次数就要减 N。如果 lfu_decay_time 取值更大，那么相应的衰减值会变小，衰减效果也会减弱。
