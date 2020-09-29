# java高级工程师、技术专家、架构师等职位面试题
 
| 序号 | 内容     | 作者      | 修改时间                |
|----|--------|---------|---------------------|
| 1  | 创建初始项目 | javajov | 2020年09月16日 |
| 2  | 增加基础内容 | javajov | 2020年09月17日 |
<br>

   
 小公司7年工作经验，仍然是个小白，认识到自己的不足，准备去大公司学习学习，但是发现招聘的软件上都是高级工程师、技术专家、架构师这样的职位，不禁在想，是大公司的兄弟们都不行连个高级工程师、技术专家、架构师都没有？还是我手机坏了，看到的都是假的东西？又或者是我飘了？但是决定了要跨出舒适区，只能硬着头皮去尝试，面试结束后，哎呀，你看那天上的月亮又大又圆，但是它为什么离我们如此遥远？果然差距很大啊，虽然多次拜读了guide哥的文章，但是发现面试的过程中仍然有很多不怎么注意的地方被问到，特地做了这个项目，仅供学习使用，写的不好的地方欢迎大家指出。

> 注：大家都熟知的经常会被问到东西不会说太多，这些去看[javaguide](https://github.com/Snailclimb/JavaGuide)哥的文章就够了，这里是对那些刁钻的或者容易忽略的问题进行补充，大家如果也有面试中的问题可以一块分享。

持续更新中...

# 目录
### 1. java篇
   - #### 基础内容
     - [ClassLoader 和 Class.forName() 的区别](https://gitee.com/javajov/java-senior-engineer-interview/blob/master/java%E5%9F%BA%E7%A1%80/classloader%E5%92%8Cclassforname.md)
     - [java 动态代理过程](https://gitee.com/javajov/java-senior-engineer-interview/blob/master/java%E5%9F%BA%E7%A1%80/%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86.md)
     - [java 8 新特性面试点](https://gitee.com/javajov/java-senior-engineer-interview/blob/master/java%E5%9F%BA%E7%A1%80/java8.md)

   - #### 集合模块
      - [HashMap](https://gitee.com/javajov/java-senior-engineer-interview/blob/master/collection/HashMap.md)
      - LinkedHashMap 
      - TreeMap 和 TreeSet
      - ConcurrentHashMap、ConcurrentSkipListMap、CopyOnWriteArrayList
   - #### 同步，并发等
      - 锁/并发控制器:synchronized，AbstractQueuedSynchronizer（AQS）、ReentrantLock、ReentrantReadWriteLock、compareAndSwap(CAS)
      - 线程池：ThreadPoolExecutor
      - ThreadLocal
   - #### jvm
     - jvm内存结构
     - java内存模型
     - 垃圾收集算法
     - 垃圾收集器（CMS和G1）
     - 类结构、类加载过程、类加载器（双亲委派模型）
     - JVM 参数，调优，OOM问题排查

2. ### 框架篇
   - #### spring
      - springMVC和springboot区别
      - spring IOC和AOP原理及过程
      - spring bean 生命周期，作用范围
      - springboot 启动过程、自动装配过程
      - 自定义注解实现过程、支持类型等
      - BeanFactory，ApplicationContext，FactoryBean区别
      - spring中常用设计模式 
   - #### mybatis
   - #### kafka，rocketMQ
     - 消息顺序性如何保证、重复消息问题、消息丢失问题
     - kafka集群原理
   - #### zookeeper
     - zookeeper 在kafka中的作用
     - 分布式锁实现的原理
     - 集群

   - #### redis
     - redis 数据类型及使用场景，布隆过滤器
     - 单线程速度快的原因、持久化机制（RDB、AOF），数据库缓存一致性问题
     - 内存淘汰机制（LRU，LFU）
     - 缓存穿透、缓存雪崩、缓存击穿
     - redis 分布式锁，锁的续期问题（redisson）
     - 主从模式、哨兵模式、集群模式（cluster）
     
   - #### elasticsearch
     - 文档检索和写入过程，删除或更新过程
     - 支持的类型，倒排序索引原理
     - 深度分页
     - 集群

3. ### 数据库篇
   - #### 索引
     - 索引结构，为什么是B+树，聚集索引，普通索引
     - sql执行过程，sql优化
   - #### 事务隔离级别
     - 并发事务的问题（脏读、幻读、不可重复读）
     - 事务的隔离击毙
     - mysql如何解决不可重复读
   - 分库分表

4. ### 微服务
   - #### dubbo
     - 组件，核心原理
     - 负载均衡策略
     - 拒绝策略
   - #### spring cloud
     - 组件，核心原理
     - 与dubbo对比
   - #### hystrix/sentinel
     - 熔断、限流、降级
5. ### 算法篇
   - LRU 算法
   - 回文字符串判断
   - 公共子串
6. ### 网络篇
   - Http，https，tcp，udp
   - tcp如何保证可靠性
   - tcp粘包问题

7. ### 其他
   - linux 常用命令
   - netty
   - IO模型
   - HBase
   - Spark
   - docker
   - Kubernetes

