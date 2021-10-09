# redis 数据结构有哪些？

redis 在线测试工具https://try.redis.io/ 如果你对redis的命令等不熟悉，可以在此测试


| 数据类型   | 说明 | 适用场景 |
|--------|----|------|
| string |  可修改的，称为动态字符串（Simple Dynamic String 简称 SDS），其内部维护一个char buf[] 数组，最大可存储512M长度的字符串，常用命令：set key value, get key, del key，incr key，decr key  |  适用与大多数KV存储结构 |
| list   |  与java中的LinkedList很像，但是结构并不相同，采用ziplist+链表的结构被称为quicklist(快速链表) ，常用命令：lpush key value1 value2 从头部添加元素，rpush key value1 value2 从尾部添加元素，llen key 列表长度，lpop key从头部弹出元素，rpop key 从尾部弹出元素，lrange key start end 查询范围内的元素,ltrim key start end 保留区间内的元素，其他都删除  | 评论列表，点赞列表，排行榜，消息队列等 |
|  hash  | 与java中的HashMap极其相似，其内部也是数组+链表的结构，常用命令：hset key field value 设置字段，hget key field 查询字段，hdel key field 删除字段 | 可以用来存储对象，购物车，商品/文章信息，评论信息等  |
|  set  | 与java中HashSet 相似，存储无序不重复的集合，常用命令：sadd  key value 添加元素，smembers key 获取所有元素，sinter key1 key2 keyN 所有集合的交集，sunion key1 key2 keyN 所有集合的并集，sdiff key1 key2 keyN 其他集合与第一个结合的差集，即第一个集合有的其他没有的  | 共同关注列表，共同好友，共同粉丝等，商品或个人标签等  |
|  zset  | 有序集合，又称SortedSet，一方面它是一个 set，保证了内部 value 的唯一性，另一方面它可以为每个 value 赋予一个 score 值，用来代表排序的权重，其内部是一个跳跃表的数据结构，常用命令：zadd key score value 添加元素，zrange key start end 获取范围内的元素  | 常用来做排行榜，电话号码簿  |


### redisObject

redis 中所有元素均由 redisObject + 其他编码方式组成，比如string格式，则实际存储的格式为：redisObject + sdshdr

```
// https://github.com/redis/redis/blob/6.2/src/server.h
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;

```

redisObject 介绍：
- type 记录对象的类型，REDIS_STRING、REDIS_LIST、REDIS_HASH、REDIS_SET、REDIS_ZSET
- encoding 对象编码：REDIS_ENCODING_EMBSTR、REDIS_ENCODING_HT、REDIS_ENCODING_ZIPLIST等
- ptr 实际存储的数据：比如，如果是string 则可能指向sdshdr结构的数据，如果是hash可能指向dict格式的数据等
- refcount 对象的引用计数信息，主要用于内存回收， 程序可以通过跟踪对象的引用计数信息， 在适当的时候自动释放对象并进行内存回收
- lru 访问时间信息，用于对象的回收

RedisObject对象很重要，Redis 对象的类型 、 内部编码 、 内存回收 、 共享对象 等功能，都是基于RedisObject对象来实现的。

### string

redis中 string 实现结构为 sdshdr

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

同样看一下hash的入口方法: hset key field value 命令入口：
```
// https://github.com/redis/redis/blob/6.2/src/t_hash.c

void hsetCommand(client *c) {
    int i, created = 0;
    robj *o;

    if ((c->argc % 2) == 1) {
        addReplyErrorFormat(c,"wrong number of arguments for '%s' command",c->cmd->name);
        return;
    }

    if ((o = hashTypeLookupWriteOrCreate(c,c->argv[1])) == NULL) return;
    // 判断是否需要将ziplist转换为hashtable 
    hashTypeTryConversion(o,c->argv,2,c->argc-1);

    // 添加或更新元素
    for (i = 2; i < c->argc; i += 2)
        created += !hashTypeSet(o,c->argv[i]->ptr,c->argv[i+1]->ptr,HASH_SET_COPY);

    /* HMSET (deprecated) and HSET return value is different. */
    char *cmdname = c->argv[0]->ptr;
    if (cmdname[1] == 's' || cmdname[1] == 'S') {
        /* HSET */
        addReplyLongLong(c, created);
    } else {
        /* HMSET */
        addReply(c, shared.ok);
    }
    signalModifiedKey(c,c->db,c->argv[1]);
    notifyKeyspaceEvent(NOTIFY_HASH,"hset",c->argv[1],c->db->id);
    server.dirty += (c->argc - 2)/2;
}
// 将ziplist转换为hashtable 
void hashTypeTryConversion(robj *o, robj **argv, int start, int end) {
    int i;

    if (o->encoding != OBJ_ENCODING_ZIPLIST) return;

    for (i = start; i <= end; i++) {
        if (sdsEncodedObject(argv[i]) &&
            sdslen(argv[i]->ptr) > server.hash_max_ziplist_value)
        {
            hashTypeConvert(o, OBJ_ENCODING_HT);
            break;
        }
    }
}
// 添加或更新元素
int hashTypeSet(robj *o, sds field, sds value, int flags) {
    int update = 0;

    if (o->encoding == OBJ_ENCODING_ZIPLIST) {
        unsigned char *zl, *fptr, *vptr;

        zl = o->ptr;
        fptr = ziplistIndex(zl, ZIPLIST_HEAD);
        if (fptr != NULL) {
            fptr = ziplistFind(zl, fptr, (unsigned char*)field, sdslen(field), 1);
            if (fptr != NULL) {
                /* Grab pointer to the value (fptr points to the field) */
                vptr = ziplistNext(zl, fptr);
                serverAssert(vptr != NULL);
                update = 1;

                /* Replace value */
                zl = ziplistReplace(zl, vptr, (unsigned char*)value,
                        sdslen(value));
            }
        }

        if (!update) {
            /* Push new field/value pair onto the tail of the ziplist */
            zl = ziplistPush(zl, (unsigned char*)field, sdslen(field),
                    ZIPLIST_TAIL);
            zl = ziplistPush(zl, (unsigned char*)value, sdslen(value),
                    ZIPLIST_TAIL);
        }
        o->ptr = zl;

        /* Check if the ziplist needs to be converted to a hash table */
        if (hashTypeLength(o) > server.hash_max_ziplist_entries)
            hashTypeConvert(o, OBJ_ENCODING_HT);
    } else if (o->encoding == OBJ_ENCODING_HT) {
        dictEntry *de = dictFind(o->ptr,field);
        if (de) {
            sdsfree(dictGetVal(de));
            if (flags & HASH_SET_TAKE_VALUE) {
                dictGetVal(de) = value;
                value = NULL;
            } else {
                dictGetVal(de) = sdsdup(value);
            }
            update = 1;
        } else {
            sds f,v;
            if (flags & HASH_SET_TAKE_FIELD) {
                f = field;
                field = NULL;
            } else {
                f = sdsdup(field);
            }
            if (flags & HASH_SET_TAKE_VALUE) {
                v = value;
                value = NULL;
            } else {
                v = sdsdup(value);
            }
            dictAdd(o->ptr,f,v);
        }
    } else {
        serverPanic("Unknown hash encoding");
    }

    /* Free SDS strings we did not referenced elsewhere if the flags
     * want this function to be responsible. */
    if (flags & HASH_SET_TAKE_FIELD && field) sdsfree(field);
    if (flags & HASH_SET_TAKE_VALUE && value) sdsfree(value);
    return update;
}

```
代码比较长，重点部分已经做了注释，我们只需要知道的是 hash结构由ziplist和hashtable两种方式实现，满足下面两个条件则使用ziplist结构：
- 对象保存的所有键值对的键和值的字符串长度都小于 64 字节
- 对象保存的键值对数量小于 512 个（可以通过配置文件调整）

ziplist 结构上面已有介绍，这里不再说明，如果是ziplist结构看一下如何存储hash内容的呢？（以下内容来自参考列表第一篇文章，强烈推荐）：

```
redis> HSET profile name "Tom"
(integer) 1
redis> HSET profile age 25
(integer) 1
redis> HSET profile career "Programmer"
(integer) 1

```
以上命令创建的profile这个key对应的值对象在redis中实际存储结构：

![输入图片说明](https://images.gitee.com/uploads/images/2021/0416/102054_e19c888a_8076629.png "屏幕截图.png")

该对象的k/v 在ziplist中的内容：

![输入图片说明](https://images.gitee.com/uploads/images/2021/0416/102200_c3f693c2_8076629.png "屏幕截图.png")

以上是hash的ziplist实现方式，图中的说明已经很清楚了，下面看一下hashtable的实现，首先看一下字典的结构定义：

```
// https://github.com/redis/redis/blob/6.2/src/dict.h
typedef struct dictEntry {
    // 字典的key
    void *key;
    union {
        // 字典的value
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    // 下一个节点，key的hash冲突时，形成了链表结构
    struct dictEntry *next;
} dictEntry;

typedef struct dictht {
    // 哈希数组table
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;

typedef struct dict {
    dictType *type;
    void *privdata;
    // 两个 dictht结构
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    int16_t pauserehash; /* If >0 rehashing is paused (<0 indicates coding error) */
} dict;
```
dict 字典中维护了 dictht 结构的两个数组，而 dictht 中保存的是dictEntry 对象，dictEntry 内部又是一个链表结构

可以看到redis中的hashtable 与java中的1.7 之前的HashMap很相似，内部都是采用数组+链表的结构存储数据。哈希冲突时，采用链地址法（拉链法）解决冲突。

![输入图片说明](https://images.gitee.com/uploads/images/2021/0416/135913_868fcf52_8076629.png "屏幕截图.png")

#### 扩容、缩容和rehash

当哈希表中的元素过多或过少时，会进行扩容和缩容，使得哈希表大小维护在一个合理的范围内：

- 正常情况下，当哈希表中的元素个数大于等于数组大小（哈希表由数组+链表组成，当总元素大于等于数组长度时就会扩容，也称为负载因子大于等于1）时就会进行扩容，新的容量为第一个大于等于总元素大小*2的值 （2 的 n 次方幂），比如当前总元素为11个，那么新的容量必须要大于11*2=22 并且时2的n 次方幂，那就是32
- 如果redis正常执行bgsave(持久化命令)操作，为了减少内存页过多分离，redis会尽力避免扩容操作，但是因为不能扩容，而哈希表中的元素又过多，超过了数组长度的5被（即负载因子大于等于5），这个时候也会进行强制扩容。容量与上面计算方式一样
- 当哈希表中的元素个数低于数组大小的10%（负载因子为0.1）时，会进行相应的缩容操作，数组长度缩减为第一个大于等于总元素大小的 2的 n 次方幂，比如当前数组大小为32，实际元素为7个，那会缩减数组长度为8

rehash过程：扩容或缩容都需要rehash过程，哈希表中用到了两个dictht ht[0] 和 ht[1]，在未进行rehash过程时，只会使用到其中的一个哈希表，为了避免 rehash 对服务器性能造成影响，redis采用的时渐进式rehash，分多次、渐进式地将 ht[0] 里面的键值对慢慢地 rehash 到 ht[1]中，rehash的详细步骤：
- 为 ht[1] 分配空间， 让字典同时持有 ht[0] 和 ht[1] 两个哈希表
- 在字典中维持一个索引计数器变量 rehashidx ， 并将它的值设置为 0 ， 表示 rehash 工作正式开始
- 在 rehash 进行期间， 每次对字典执行添加、删除、查找或者更新操作时， 程序除了执行指定的操作以外， 还会顺带将 ht[0] 哈希表在 rehashidx 索引上的所有键值对 rehash 到 ht[1] ， 当 rehash 工作完成之后， 程序将 rehashidx 属性的值增一
- 随着字典操作的不断执行， 最终在某个时间点上， ht[0] 的所有键值对都会被 rehash 至 ht[1] ， 这时程序将 rehashidx 属性的值设为 -1 ， 表示 rehash 操作已完成

渐进式 rehash 的好处在于它采取分而治之的方式， 将 rehash 键值对所需的计算工作均滩到对字典的每个添加、删除、查找和更新操作上， 从而避免了集中式 rehash 而带来的庞大计算量。

rehash 期间的操作会同时使用 ht[0] 和 ht[1] 两个哈希表，字典的删除（delete）、查找（find）、更新（update）等操作会在两个哈希表上进行， 新添加到字典的键值对一律会被保存到 ht[1] 里面。

最终rehash完成后，会释放 ht[0] ， 将 ht[1] 设置为 ht[0] ， 并在 ht[1] 新创建一个空白哈希表， 为下一次 rehash 做准备


参考：

[哈希对象](http://redisbook.com/preview/object/hash.html)

[rehash](http://redisbook.com/preview/dict/rehashing.html)

[渐进式 rehash](http://redisbook.com/preview/dict/incremental_rehashing.html)

[Redis(1)——5种基本数据结构](https://www.wmyskxz.com/2020/02/28/redis-1-5-chong-ji-ben-shu-ju-jie-gou/)

### set

java 中的set 是一种无序不重复的集合，redis中也是一样，redis中的set也有两种实现方式，满足以下两个条件时采用intset 存储，否则，采用哈希表hashtable（只使用hash表中的key添加元素，value为null，这一点也与java中的HashSet相似） ：

- 保存的所有元素都是整数值
- 保存的元素数量不超过512个

哈希表结构，在上一节已经介绍过，这里不再详细说明，下面针对intset做以下简单的介绍：

```
// https://github.com/redis/redis/blob/6.2/src/intset.h
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
```

内部其实是一个int数组

![输入图片说明](https://images.gitee.com/uploads/images/2021/0416/140248_796728ac_8076629.png "屏幕截图.png")

#### 使用场景说明

set的特性时无序不重复，redis中提供了交、并、差集的方法，根据这些方法可以实现共同好友、共同关注的人等，也可以用来做数据去重等。


参考：

[redis set底层数据结构](https://www.jianshu.com/p/28138a5371d0)

[集合对象](http://redisbook.com/preview/object/set.html)

### zset

zset 又称sortedset，它是一个有序不重复的集合。在原有set的基础上提供了一个score作为排序。zset也是有两种实现方式，满足以下两个条件时使用的是ziplist 否则使用skiplist 

- 保存的元素数量小于 128 个
- 保存的所有元素成员的长度都小于 64 字节

使用 ziplist 时 每个集合元素使用两个紧挨在一起的压缩列表节点来保存， 第一个节点保存元素的成员， 而第二个元素则保存元素的分值（score），例子：

```
redis> ZADD price 8.5 apple 5.0 banana 6.0 cherry
(integer) 3

```
![输入图片说明](https://images.gitee.com/uploads/images/2021/0416/145354_f6c158cc_8076629.png "屏幕截图.png")


使用skiplist （跳跃表）时，源码中的实现

```
// https://github.com/redis/redis/blob/6.2/src/server.h

typedef struct zskiplistNode {
    sds ele;
    double score;
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;

typedef struct zset {
    dict *dict;
    zskiplist *zsl;
} zset;

```
zset 由哈希表 + zskiplist 实现

zskiplist ：

- header，tail 指向跳跃表的表头和表尾节点
- level 记录目前跳跃表内，层数最大的那个节点的层数
- length 跳跃表目前包含节点的数量

zskiplistNode：
- 层（level）：节点中用 L1 、 L2 、 L3 等字样标记节点的各个层， L1 代表第一层， L2 代表第二层，以此类推。每个层都带有两个属性：前进指针和跨度。前进指针用于访问位于表尾方向的其他节点，而跨度则记录了前进指针所指向节点和当前节点的距离。
- 后退（backward）指针：节点中用 BW 字样标记节点的后退指针，它指向位于当前节点的前一个节点。后退指针在程序从表尾向表头遍历时使用
- 分值（score）：各个节点中的 1.0 、 2.0 和 3.0 是节点所保存的分值。在跳跃表中，节点按各自所保存的分值从小到大排列

![输入图片说明](https://images.gitee.com/uploads/images/2021/0416/155138_f5c5f0d9_8076629.png "屏幕截图.png")

最后看一下整体存储结构：

redis服务器默认会创建16个数据库，所有的键值对都保存在dict字典中。

![输入图片说明](https://images.gitee.com/uploads/images/2021/0421/105513_88af5b83_8076629.png "屏幕截图.png")


参考：

[有序集合对象](http://redisbook.com/preview/object/sorted_set.html)

[跳跃表的实现](http://redisbook.com/preview/skiplist/datastruct.html)

[Redis整体数据存储说明](https://itzones.cn/2020/04/30/Redis%E6%95%B4%E4%BD%93%E6%95%B0%E6%8D%AE%E5%AD%98%E5%82%A8%E8%AF%B4%E6%98%8E/)

### 如何批量删除key？

如果我们知道key的名称，可以通过 del key1 key2 key3 删除多个key，不存在的话会忽略

如果我们只知道key的前缀或者某个特殊符号等，可以使用 scan + xargs 命令删除（当然另外也有 keys+xargs 命令可以删除但是实际生产环境中并不推荐，这里也就不介绍了）：

```
./redis-cli -h 127.0.0.1 -p 6379 --scan --pattern 'java*' | xargs ./redis-cli -h 127.0.0.1 -p 6379 del

//异步删除 把del 换成unlink

./redis-cli -h 127.0.0.1 -p 6379 --scan --pattern 'java*' | xargs ./redis-cli -h 127.0.0.1 -p 6379 unlink

```

### redis中的 布隆过滤器

什么是布隆过滤器？

它是一种数据结构，是由一串很长的二进制向量组成，可以将其看成一个二进制数组或者位数组，数组中的元素只有0和1两种值。

当我们将某个放入布隆过滤器时，要经过多个哈希函数，计算哈希值在数组中的存放位置，将对应的位置从0 设置为 1

例如，图中数字代表布隆过滤器的数组，经过3个哈希函数，Hello元素对应的不同位置变为了1：

![输入图片说明](https://images.gitee.com/uploads/images/2021/0420/144011_ebff59fe_8076629.png "屏幕截图.png")

布隆过滤器是用于判断一个元素是否在集合中。通过一个位数组和N个hash函数实现

优点：

空间效率高，所占空间小。查询时间短。

缺点：

元素添加到集合中后，不能被删除。有一定的误判率


布隆过滤器最大的特点是，如果它判断某个值不存在，那么一定不存在，如果它判断存在，有可能误判，比如布隆过滤器中对应的位置被其他元素先占用了等。


基于此，我们可以用来判断某些key是不是存在，防止缓存穿透的情况。

redis 中我们经常会做一些热点数据为缓存，当请求过来之后先查缓存，缓存没有的时候再去查数据库，如果是正常的请求，都能从缓存中查询到，这样也提高了我们的响应速度，可是，如果请求的参数，不是我们系统中存在的，redis中一定没有的，那么这些请求就将全部落到了数据库上，给我们的数据库造成了很大的压力，甚至造成数据库崩溃的情况。

针对这种情况，我们可以先把系统中的数据先在布隆过滤器中放置一遍，当请求过来的时候，先在布隆过滤器中判断是否存在，不存在的直接返回，存在之后再判断缓存和数据库，这样就可以过滤调大部分的非正常请求，从而防止缓存穿透的情况。

redis 中的布隆过滤器实现：

```
// 方案一 还有更多的细节需要实现，比如hash函数的实现，计算对应的offset等
SETBIT key offset value:对key设置指定offset上的值
GETBIT key offset:获取key指定offset上对应的值

// 方案二 使用RedisBloom https://github.com/RedisBloom/RedisBloom 更多细节到这里查看
BF.ADD newFilter foo
BF.EXISTS newFilter foo

// 方案三 使用Redisson（https://github.com/redisson/redisson） 这是java版的一个封装好的redis类库，里面不仅有布隆过滤器的实现，还有redis的分布式锁等，该库会在另一个章节中介绍
RBloomFilter<String> bloomFilter = redisson.getBloomFilter("phoneList");
//初始化布隆过滤器：预计元素为100000000L,误差率为3%
bloomFilter.tryInit(100000000L,0.03);
//将号码10086插入到布隆过滤器中
bloomFilter.add("10086");
//判断下面号码是否在布隆过滤器中
System.out.println(bloomFilter.contains("123456"));//false
System.out.println(bloomFilter.contains("10086"));//true

```

参考：

[Redis(5)——亿级数据过滤和布隆过滤器](https://www.wmyskxz.com/2020/03/11/redis-5-yi-ji-shu-ju-guo-lu-he-bu-long-guo-lu-qi/)

[Redis详解（十三）------ Redis布隆过滤器](https://www.cnblogs.com/ysocean/p/12594982.html)

### redis中的 geo

Redis基于[geohash](https://redis.io/commands/geohash)和有序集合(zset)提供了地理位置相关功能，可以用来实现查找附近的人，附近的共享单车，附近的XXX等

geohash的思想是将二维的经纬度转换成一维的字符串，geohash有以下三个特点:
- 字符串越长，表示的范围越精确
- 字符串相似的表示距离相近，利用字符串的前缀匹配，可以查询附近的地理位置。这样就实现了快速查询某个坐标附近的地理位置。
- geohash计算的字符串，可以反向解码出原来的经纬度

更多细节请参考这篇文章[GeoHash原理及redis geo相关操作](https://segmentfault.com/a/1190000038529554)

### redis中的HyperLogLog

HyperLogLog 是一种基数估算算法，所谓基数估算，就是估算在一批数据中，不重复元素的个数有多少，HyperLogLog 的优点是，在输入元素的数量或者体积非常大时，计算基数所需的空间总是固定的、并且是很小的。常用于统计日活、月活等数据

redis 中 提供的命令：

```
# 用于向 HyperLogLog 添加元素
# 如果 HyperLogLog 估计的近似基数在 PFADD 命令执行之后出现了变化， 那么命令返回 1 ， 否则返回 0 
# 如果命令执行时给定的键不存在， 那么程序将先创建一个空的 HyperLogLog 结构， 然后再执行命令
pfadd key value1 [value2 value3]

# PFCOUNT 命令会给出 HyperLogLog 包含的近似基数
# 在计算出基数后， PFCOUNT 会将值存储在 HyperLogLog 中进行缓存，知道下次 PFADD 执行成功前，就都不需要再次进行基数的计算。
pfcount key

# PFMERGE 将多个 HyperLogLog 合并为一个 HyperLogLog ， 合并后的 HyperLogLog 的基数接近于所有输入 HyperLogLog 的并集基数。
pfmerge destkey key1 key2 [...keyn]

```

参考：

[Redis 中 HyperLogLog 的使用场景](https://www.cnblogs.com/54chensongxia/p/13803465.html)

