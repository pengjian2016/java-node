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

底层实现，扩容过程

#### list

#### hash

底层实现，rehash过程

#### set

#### zset

#### redis中的 布隆过滤器

#### redis中的 geo

#### redis中的HyperLogLog


