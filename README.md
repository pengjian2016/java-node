# java高级工程师面试
 
| 序号 | 内容     | 作者      | 修改时间                |
|----|--------|---------|---------------------|
| 1  | 创建初始项目 | javajov | 2020年09月16日17:16:06 |





   写在前面的话，毕业之后进入了一家中日合资的企业，工作上主要就是维护日企的CMS管理系统，没有使用任何的框架，就单纯的java web应用，jsp，servlet这样的硬撸，数据库控制这块都是自己写的Class.forName加载数据库驱动，创建Connection，打开session这种，当时给我的感觉就是在这个公司再待下去就废了，这些玩意完全就是学校里做实验时的demo啊，我能学到啥东西啊，于是半年之后就果断离职。不知道听谁说的去小公司能学到更多东西，我信了，投了一家小公司，面试结束之后就收到了offer，然后回家过年，年后入职，当时心里美滋滋，呵呵。

   这一声‘呵呵’，不知道隐藏了多少辛酸，小公司学的东西多我是真的体会到了，前端UI自己设计，后端服务自己搭，逻辑自己写，数据库表自己设计，开发完了之后自己测试一遍，没啥问题，可以给客户部署了。啥？服务器要我自己装？《centos7安装教程》就按这个装了，哎呀，怎么下载不了东西啊，应用都已经运行了，怎么访问不了啊？也没啥错误啊，配dns干嘛？iptable是啥？hosts修改有啥用？firewall啥玩意啊？80端口？在线等挺急的。

  说这么多，其实不是抱怨什么，只是当时真的不懂这些东西，也不会有人教你，只能自己学习。后面还开发过android应用，ios应用，写过python，甚至用js写过桌面应用程序，因为公司需要，因为客户说要有，所以我们都会满足他。刚开始的两三年基本上就是东搞一下，西搞一下，基本上是有什么需求做什么需求，没有固定的项目。后面自己负责的其中一个项目算是做起来了，开始带一两个人的小团队，但我一直没有把自己当个项目组长，把他们当成自己的兄弟，来需求了都是你分一点，我分一点，代码也没有严格要求，功能测试一下没啥问题就ok了，然而等他们离职之后，我意思到了问题，上线的客户反馈了一大堆问题，我花了大半年的时间在填坑。这个时候我开始反思自己，适合做这个Leader吗，有资格吗，后面又招的人我比他们强吗？我觉得有必要深入学习了，先去找了面试题，看看大公司主要关注哪一块，在github上搜java，排在第一位的就是guide哥的项目，看了他这里面的面试关注点，我才意识到我这几年基本上是在浪费时间。认真的花了两个月的时间读了每一篇文章，写的真的是不错啊，一下子让我这种空瓶子装满了水的感觉，以前那些不太明白的地方，豁然开朗了。可是回到项目中那些东西我仍然没办法融入到我自己的项目中去，我不太明白为什么研究的这么深入，像HashMap，我在项目里无非就是存放一些简单的key，value值，把它返回到页面或者保存到库中，可是研究它的扩容机制，研究它的rehash过程是为了解决什么样的场景问题呢？像是ConcurrentHashMap 这种多线程场景下的集合，我们这种单机的项目，要保存什么数据才会有多线程并发呢？在我理解我们这种Web应用，每个请求都是一个独立的线程，有这种共享数据，我直接用数据库，用redis不行吗？还有就是像synchronized，ReentrantLock，ThreadPoolExecutor这些在我们项目中完全没用到，我们是不是太low了，是不是我接触的层次不太够，或者这些东西我还没理解透所以没办法融入到项目中，于是我又花了3个月的时间，仔细去看了guide哥的宝典，读完之后仍然没有什么作用，这个时候我已经有迫切的想要离职的想法了，我想要进入大公司看看他们到底怎么用的，到底我跟他们有多大的差距，本来计划2020年后换工作，然而由于疫情的原因，我负责的这个项目本来不瘟不火的一下子增加了40多个客户，定制需求一个接一个来，上半年基本上没有什么休息的时间，我也搁置了原先的计划，等忙完了再说，在此期间我又去看了一遍guide哥的文章，对于我这样的没啥技术，只知道为客户定制需求，这么多年没有经过线上问题洗礼的小白来讲，真的是宝典了，这里面太多我不知道的东西了。8月下旬的时候，刚从客户现场那里出差回来，我决定跨出舒适区，第一件事就是到直聘上准备了自己的简历，投递了第一份简历，第二天就约了面试，然后就被虐的体无完肤，惭愧啊，丢了java程序员的脸。但我知道不能放弃，因为在这里，再过十年我也只能是比小白少强一点的小白。

  回顾这7年来，感觉真的是废。现在开始一边学习一边慢慢的面试，基本上投的都是大公司，小公司怕了啊，但是收到的面试机会却是比较少，可能履历有点差吧，面试的时候发现大多都是高级工程师，技术专家，架构师这样的职位，说实话我是有点虚的，我怕是个中级都不到啊，但是硬着头皮投了，显然是没有结果啊，然后就发现guide哥的面试宝典只适合1~3年的人，像我这样7年以上的，那些基础完全不问啊，竟是挑些角度刁钻的问题问，所以特地搞了这个项目，收集整理高级工程师，技术专家，架构师这方面的面试题，供大家参考和自己学习使用，写的不好的地方请见谅。