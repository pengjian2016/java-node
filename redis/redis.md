# redis 数据结构有哪些？

redis 在线测试工具https://try.redis.io/ 如果你对redis的命令等不熟悉，可以在此测试


| 数据类型   | 说明 | 适用场景 |
|--------|----|------|
| string |  可修改的，称为动态字符串（Simple Dynamic String 简称 SDS），其内部维护一个char buf[] 数组，最大可存储512M长度的字符串，常用命令：set key value, get key, del key，incr key，decr key  |  适用与大多数KV存储结构 |
| list   |  与java中的LinkedList很像，但是结构并不相同，采用ziplist+链表的结构被称为quicklist(快速链表) ，常用命令：lpush key value1 value2 从头部添加元素，rpush key value1 value2 从尾部添加元素，llen key 列表长度，lpop key从头部弹出元素，rpop key 从尾部弹出元素，lrange key start end 查询范围内的元素,ltrim key start end 保留区间内的元素，其他都删除  | 评论列表，点赞列表，排行榜，消息队列等 |
|  hash  | 与java中的HashMap极其相似，其内部也是数组+链表的结构，常用命令：hset key field value 设置字段，hget key field 查询字段，hdel key field 删除字段 | 可以用来存储对象，购物车，商品/文章信息，评论信息等  |
|  set  | 与java中HashSet 相似，存储无序不重复的集合，常用命令：sadd  key value 添加元素，smembers key 获取所有元素，sinter key1 key2 keyN 所有集合的交集，sunion key1 key2 keyN 所有集合的并集，sdiff key1 key2 keyN 其他集合与第一个结合的差集，即第一个集合有的其他没有的  | 共同关注列表，共同好友，共同粉丝等，商品或个人标签等  |
|  zset  | 有序集合，又称SortedSet，一方面它是一个 set，保证了内部 value 的唯一性，另一方面它可以为每个 value 赋予一个 score 值，用来代表排序的权重，其内部是一个跳跃表的数据结构，常用命令：zadd key score value 添加元素，zrange key start end 获取范围内的元素  | 常用来做排行榜，电话号码簿  |


### string

redis中 string 实际的结构为 redisObject+sdshdr

redisObject 由：type、encoding、ptr组成

RedisObject对象很重要，Redis 对象的类型 、 内部编码 、 内存回收 、 共享对象 等功能，都是基于RedisObject对象来实现的。

sdshdr 的结构，官方文档（https://redis.io/topics/internals-sds）：

```
struct sdshdr {
    long len;
    long free;
    char buf[];
};

```

查看官方的源码看到的是下面这样：
```
源码：https://github.com/redis/redis/blob/6.2/src/sds.h
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};

```
虽然有点差异，但是总体原理是差不多的，都是内部维护了一个char 数组，有点和java中的StringBuilder相似。

根据不同的类型和长度，string又有三种编码方式：

- 如果一个字符串对象保存的是整数值， 用 int 编码方式 ，字符串对象会将整数值保存在字符串对象结构的 ptr 属性里面（将 void* 转换成 long ），也就是没有sdshdr部分

- 如果符串长度小于等于44，则用embstr编码，redisObject+sdshdr

- 当字符串长度大于44，则用raw编码，redisObject+sdshdr

为什么 44 以下用embstr 而不是直接都用raw呢？
- embstr 编码将创建字符串对象所需的内存分配次数从 raw 编码的两次降低为一次
- 释放 embstr 编码的字符串对象只需要调用一次内存释放函数， 而释放 raw 编码的字符串对象需要调用两次内存释放函数
- 因为 embstr 编码的字符串对象的所有数据都保存在一块连续的内存里面， 所以这种编码的字符串对象比起 raw 编码的字符串对象能够更好地利用缓存带来的优势

说白了就是为了提高性能

下图为 redisObject+sdshdr 的结构：

![string](https://images.gitee.com/uploads/images/2021/0413/142125_d5229882_8076629.png "stringj结构.png")

如果上面这个不够直观，再看下面这个，当我们调用 set key value 命令时（省略了其他信息），实际的存储结构为：

![输入图片说明](https://images.gitee.com/uploads/images/2021/0413/143129_0614b909_8076629.png "屏幕截图.png")

这样设计的好处是什么呢？（为什么要用三种编码方式）？

可以针对不同的使用场景，设置不同的类型，从而优化对象在不同场景下的使用效率。

sdshdr 扩容策略：

- 如果sds字符串修改之后，新长度小于1M，扩容和len等长的未分配空间。比如修改之后为13个字节，那么程序也会分配 13 字节的未使用空间
- 新长度大于等于1M，每次扩容变为增加1Mb

sdshdr 缩容和惰性空间释放：

- 当某些操作后，减少了原有字符串大小，sds并不会立即重新释放空间，重新分配内存，而是继续保留那么多空间，说不定下次就用上了，它会修改len

c语言中也有字符串，为什么不直接使用，而是新定义了sdshdr ？

- sdshdr中存储字符串长度，获取字符长度只需要O(1)时间复杂度，而原生字符串则需要遍历数组。
- 杜绝缓冲区溢出，C语言字符串本身不记录数组长度，增加操作时，分配内存不足时容易造成缓冲区溢出，而sds因为存在alloc，会在修改时，检查空间大小是否满足
- 内存预分配以及惰性删除，减少内存重新分配次数
- 二进制安全，可以存储字节数据，因为存储字符串长度，不会提前遇到\0字符而终止
- 兼容C标准库中的部分字符串函数.

redis在系统中的角色越来越重要，面试也越来越深入，我们需要清楚它的存储结构和扩容方式，这样在实际开发中，能明白哪些因素会影响它的性能。


参考：

[字符串对象](http://redisbook.com/preview/object/string.html)

[一个简单的字符串，为什么 Redis 要设计的如此特别](https://www.cnblogs.com/lonely-wolf/p/14261486.html)

[深入浅出Redis之sds](http://wzmmmmj.com/2020/07/12/redis-sds/)


### list

从源码中看一下list的入口方法：

```
//list lpush 命令：https://github.com/redis/redis/blob/6.2/src/t_list.c
void lpushCommand(client *c) {
    pushGenericCommand(c,LIST_HEAD,0);
}
void pushGenericCommand(client *c, int where, int xx) {
    int j;

    robj *lobj = lookupKeyWrite(c->db, c->argv[1]);
    if (checkType(c,lobj,OBJ_LIST)) return;
    if (!lobj) {
        if (xx) {
            addReply(c, shared.czero);
            return;
        }
        // 创建quicklist对象
        lobj = createQuicklistObject();
        quicklistSetOptions(lobj->ptr, server.list_max_ziplist_size,
                            server.list_compress_depth);
        dbAdd(c->db,c->argv[1],lobj);
    }

    for (j = 2; j < c->argc; j++) {
        listTypePush(lobj,c->argv[j],where);
        server.dirty++;
    }

    addReplyLongLong(c, listTypeLength(lobj));

    char *event = (where == LIST_HEAD) ? "lpush" : "rpush";
    signalModifiedKey(c,c->db,c->argv[1]);
    notifyKeyspaceEvent(NOTIFY_LIST,event,c->argv[1],c->db->id);
}

//quicklist定义：https://github.com/redis/redis/blob/6.2/src/quicklist.h
// quicklistNode is a 32 byte struct describing a ziplist for a quicklist
typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl;
    unsigned int sz;             /* ziplist size in bytes */
    unsigned int count : 16;     /* count of items in ziplist */
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
    unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */
    unsigned int recompress : 1; /* was this node previous compressed? */
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;

typedef struct quicklistLZF {
    unsigned int sz; /* LZF size in bytes*/
    char compressed[];
} quicklistLZF;
typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;        /* total count of all entries in all ziplists */
    unsigned long len;          /* number of quicklistNodes */
    int fill : QL_FILL_BITS;              /* fill factor for individual nodes */
    unsigned int compress : QL_COMP_BITS; /* depth of end nodes not to compress;0=off */
    unsigned int bookmark_count: QL_BM_BITS;
    quicklistBookmark bookmarks[];
} quicklist;

```

list数据类型是由quicklist 实现，而quicklist 内部维护了quicklistNode 结构的头节点和尾节点，看一下quicklistNode 内部结构：
- prev，next 前驱和后续节点的指针
- zl 实际存储的数据，如果当前节点的数据没有压缩，那么它指向一个ziplist结构；否则，它指向一个quicklistLZF结构
- sz: 表示zl指向的ziplist的总大小（包括zlbytes, zltail, zllen, zlend和各个数据项）。需要注意的是：如果ziplist被压缩了，那么这个sz的值仍然是压缩前的ziplist大小。
- count: 表示ziplist里面包含的数据项个数。这个字段只有16bit
- encoding: 表示ziplist是否压缩了，2表示被压缩了，1表示没有压缩。其他字段感兴趣的可以看参考中的第一篇文章

如果压缩了，则zl会指向quicklistLZF结构，quicklistLZF的源码中：
- sz: 表示压缩后的ziplist大小。
- compressed: 是个柔性数组，存放压缩后的ziplist字节数组。

quicklistNode结构中，实际存储数据的zl指针，实际上是一个ziplist（稍后介绍），到这里我们能看出quicklist的整体结构，双向链表+ziplist

![输入图片说明](https://images.gitee.com/uploads/images/2021/0414/155240_c855618b_8076629.png "屏幕截图.png")

为什么要这么设计呢？为什么不直接使用双向链表或者使用ziplist来存储，而是要包装一层呢？

在回答这个问题之前，我们先看一下ziplist，官方源码中ziplist 不像其他数据结构有具体的类型，在官方的注解中有这样一段描述：

```
// https://github.com/redis/redis/blob/6.2/src/ziplist.c
ziplist 结构描述
The general layout of the ziplist is as follows:
<zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>

entry 描述：
So a complete entry is stored like this:
<prevlen> <encoding> <entry-data>

```
- zlbytes：32bit 表示ziplist占用的字节总数，其中包含自己本身占用的4个字节
- zltail：32bit 表示ziplist 最后一个entry 距离列表起始地址有多少个字节。它的存在意味可以很方便地找到最后一项，而不需要遍历整个列表，并且可以快速的进行pop和push操作
- zllen：16bit 表示entry的个数
- entry :表示真正存放数据的数据项
- zlend ：8bit ，ziplist最后一项，表示结尾，固定值255

下面是entry 说明：
- prevlen 表示前一个数据项占用的总字节数，可以方便从后向前遍历。
- encoding 表示当前数据项所保存数据的类型以及长度
- entry-data 实际的数据

ziplist 使用的是一块连续的内存，它要比普通的链表结构更加节省内存（就以java中的LinkedList来说，链表中每一项都占用独立的一块内存，各项之间用引用连接起来，这种方式会带来大量的内存碎片，而且引用也会占用额外的内存），同时也减少了内存碎片。但是因为ziplist的复杂结构，也让它变的不利于修改。

回到上面的问题，为什么quicklist 要设计成双向链表+ziplist？

这也是空间和时间的折中：
- 双向链表便于在两端进行插入和删除，但是需要额外的空间存储前驱和后续节点的指针或引用，链表中的每个节点都是独立的，空间不连续，节点多了容易产生内存碎片。
- ziplist由于是一整块连续内存，所以存储效率很高，但是不利于修改，每次数据变动都会引发一次内存的realloc，特别是数据很长的时候，使得它变得效率低下。

结合二者的优点，设计出了quicklist 

那么一个新的元素是加入到当前节点的ziplist中，还是新建节点放入，如何规定ziplist的长度呢？

Redis提供了一个配置参数list-max-ziplist-size：

- 当取正值的时候，表示按照数据项个数来限定每个quicklist节点上的ziplist长度。比如：当这个参数配置成5的时候，表示每个quicklist节点的ziplist最多包含5个数据项
- 当取负值的时候，表示按照占用字节数来限定每个quicklist节点上的ziplist长度，这个时候只能取5个值：
- -5: 每个quicklist节点上的ziplist大小不能超过64 Kb
- -4: 每个quicklist节点上的ziplist大小不能超过32 Kb
- -3: 每个quicklist节点上的ziplist大小不能超过16 Kb
- -2: 每个quicklist节点上的ziplist大小不能超过8 Kb（默认值，在官方的配置文件redis.conf中，推荐-2和-1）
- -1: 每个quicklist节点上的ziplist大小不能超过4 Kb 


#### 使用场景

评论列表如何实现？

上面list的底层实现原理我们已有了解，知道它可能不太适合大记录的存储，一条评论信息包含评论id，评论人信息，评论内容等，我们一般不会把整条评论序列化之后存储到list中，
而是以文章id为key，评论id为value放入到list中，评论的详细信息通过数据库或者redis的hash结构再次查询获得。关注列表、点赞列表等也是类似的逻辑。

list如何实现消息队列？

list 提供的 lpush/rpush和lpop/rpop 所以我们可以在头部插入，尾部弹出消息。

但是如果只用lpush rpop这样的命令处理消息，会存在一个性能风险点，比如消费者如果想要及时的处理数据，就要在程序中写个类似 while(true) 这样的循环，不停的去调用 RPOP 或 LPOP 命令，这就会给消费者程序带来些不必要的性能损失

好在redis 提供了BLPOP、BRPOP 这种阻塞式读取的命令，客户端在没有读到队列数据时，自动阻塞，直到有新的数据写入队列，再开始读取新数据。这种方式就节省了不必要的 CPU 开销。

所以list实现消息队列主要通过，lpush/rpush blpop/brpop 这些命令。

当然redis中还有 Publish/Subscribe 消息的发布和订阅模式，提供消息队列的能力，这里暂时不详细展开。

参考:

[Redis内部数据结构详解(5)——quicklist](http://zhangtielei.com/posts/blog-redis-quicklist.html)

[Redis 存储效率的追求-ziplist](https://www.jianshu.com/p/08da62e21fa5)


### hash

底层实现，rehash过程

### set

### zset

### 如何批量删除key？

### redis中的 布隆过滤器

### redis中的 geo

### redis中的HyperLogLog
