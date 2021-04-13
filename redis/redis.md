# redis 数据结构有哪些？

redis 在线测试工具https://try.redis.io/ 如果你对redis的命令等不熟悉，可以在此测试


| 数据类型   | 说明 | 适用场景 |
|--------|----|------|
| string |  可修改的，称为动态字符串（Simple Dynamic String 简称 SDS），其内部维护一个char buf[] 数组，最大可存储512M长度的字符串，常用命令：set key value, get key, del key，incr key，decr key  |  适用与大多数KV存储结构 |
| list   |  与java中的LinkedList很像，但是结构并不相同，采用ziplist+链表的结构被称为quicklist(快速链表) ，常用命令：lpush key value1 value2 从头部添加元素，rpush key value1 value2 从尾部添加元素，llen key 列表长度，lpop key从头部弹出元素，rpop key 从尾部弹出元素，lrange key start end 查询范围内的元素,ltrim key start end 保留区间内的元素，其他都删除  | 评论列表，点赞列表，排行榜，消息队列等 |
|  hash  | 与java中的HashMap极其相似，其内部也是数组+链表的结构，常用命令：hset key field value 设置字段，hget key field 查询字段，hdel key field 删除字段 | 可以用来存储对象，购物车，商品/文章信息，评论信息等  |
|  set  | 与java中HashSet 相似，存储无序不重复的集合，常用命令：sadd  key value 添加元素，smembers key 获取所有元素，sinter key1 key2 keyN 所有集合的交集，sunion key1 key2 keyN 所有集合的并集，sdiff key1 key2 keyN 其他集合与第一个结合的差集，即第一个集合有的其他没有的  | 共同关注列表，共同好友，共同粉丝等，商品或个人标签等  |
|  zset  | 有序集合，又称SortedSet，一方面它是一个 set，保证了内部 value 的唯一性，另一方面它可以为每个 value 赋予一个 score 值，用来代表排序的权重，其内部是一个跳跃表的数据结构，常用命令：zadd key score value 添加元素，zrange key start end 获取范围内的元素  | 常用来做排行榜，电话号码簿  |


#### string

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

- 当某些操作后，减少了原有字符串大小，sds并不会立即重新释放空间，重新分配内存，而是继续保留那么多空间，说不定下次就用上了。它会修改len

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


#### list

#### hash

底层实现，rehash过程

#### set

#### zset

#### 如何批量删除key？

#### redis中的 布隆过滤器

#### redis中的 geo

#### redis中的HyperLogLog
