### 倒排序原理

Elasticsearch 是通过 Lucene 的倒排序索引技术实现的。

在传统没有索引的时候，我们先找到一篇文章，然后获取里面的单词，这种就是所谓的”正向索引”，后来，我们希望能够输入一个单词，找到含有这个单词，或者和这个单词有关系的文章于是就把这种索引"倒排索引”，通俗地来讲，正向索引是通过key找value，反向索引则是通过value找key。

#### 基本概念

-  **Term（单词）** ：一段文本经过分析器分析以后就会输出一串单词，这一个一个的就叫做Term
-  **Term Dictionary（单词字典）** ：顾名思义，它里面维护的是Term，可以理解为Term的集合
-  **Term Index（单词索引）** ：为了更快的找到某个单词，我们为单词建立索引
-  **Posting List（倒排列表）** ：倒排列表记录了出现过某个单词的所有文档的文档列表及单词在该文档中出现的位置信息，每条记录称为一个倒排项(Posting)。根据倒排列表，即可获知哪些文档包含某个单词

假设有下面这样的三条记录：

| id | name | age |
|----|------|-----|
| 1  | 张三   | 20  |
| 2  | 李四   | 21  |
| 3  | 王五   | 20  |

存放的Elasticsearch中，它分别为每个字段都建立了一个倒排索引，大致结果如下：

name 字段：

| Term | Posting List |
|------|--------------|
| 张三   | 1            |
| 李四   | 2            |
| 王五   | 3            | 

age 字段：

| Term | Posting List |
|------|--------------|
| 20   | [1,3]            |
| 21   | 2            |

比如要检索age为20的所有人员，那么会从 age的Term Dictionary中找到[1,3]两条id，然后根据id获取详细的信息。

然而单词是非常多的，单词字典就会变的非常大，在这样的Term Dictionary中如何快速找到我们需要的关键字呢，没错就是索引（Term Index）

![输入图片说明](https://images.gitee.com/uploads/images/2021/1026/100241_3dc9c7ac_8076629.png "屏幕截图.png")

Term Index： 通过Trie树（前缀树/字典树，类似与我们的汉语词典，通过开头的字母定位到大致的页数）存储Term的某些前缀，并会对Term Index进行某些压缩。

把Term Index加载到内存后，可快速查找到Term在Term Dictionary中的大致位置，再利用Term Dictionary在磁盘上二分查找目标Term。这已经大大减少了直接对Term Dictionary在磁盘进行二分查找带来的随机IO操作消耗。

Term index实际上在内存中是以FST（finite state transducers）的形式保存的，其特点是非常节省内存。可以在FST上实现单Term、Term范围、Term前缀和通配符查询等。

总结倒排序索引原理：通过对对关键字进行分词，组成一张单词字典表，记录包含该单词的文档ID，并为单词字典建立索引，搜索的过程使用  term index -> term dictionary -> postings list  的倒排索引结构，快速定位搜索单词对应的文档id。

#### 倒排序索引为什么比 数据库的B+树索引快？

Mysql 只有 term dictionary 这一层，是以 b-tree 排序的方式存储在磁盘上的。检索一个 term 需要若干次随机 IO 的磁盘操作。而 Lucene 在 term dictionary 的基础上添加了term index来加速检索，term index 以树的形式缓存在内存中。从 term index 查到对应的 term dictionary 的 block 位置之后，再去磁盘上找 term，大大减少了磁盘的 random access （随机IO）次数。

#### mysql为什么不用倒排序索？
倒排序索引快的同时，也牺牲了实时性，倘若某条记录写入后几秒钟才能被检索到，就不得不考虑是不是适合我们的业务场景了，当然通过某种手段是可以保证试更新完倒排序索的同时能够被检索到，但是这样的话，写入的吞吐量就减低了。


关于倒排序这一块其实还有很多的知识点要介绍，比如如何构建索引的，如何更新，哪些地方用到了压缩，什么压缩算法？多条件联合查询的过程等，感兴趣的可以去看一下。

> 有的时候觉得写的过于详细，以至于文章不像是在写面试的东西，更像是在探讨技术细节，但是一笔概括的话，又很难让人理解，我也在思考这个问题。


参考：

[Elasticsearch 倒排索引原理 |](https://xiaoming.net.cn/2020/11/25/Elasticsearch%20%E5%80%92%E6%8E%92%E7%B4%A2%E5%BC%95/)

[Lucene 倒排索引原理](https://juejin.cn/post/6947984489960177677)

[Elasticsearch 如何做到快速检索 - 倒排索引的秘密](https://segmentfault.com/a/1190000037658997)