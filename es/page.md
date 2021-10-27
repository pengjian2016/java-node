### Elasticsearch 深度分页的几种方式

#### from + size

这是Elasticsearch默认采用的分页方式，虽然有点类似数据库的分页方式，但是实际做法却非常耗内存

```
GET /student/student/_search
{
  "query":{
    "match_all": {}
  },
  "from":990,
  "size":10
}

```

假如查询： from = 990 , size = 10 , 分片数为：4，es会在每个分片获取1000条文档，通过协调节点(Coordinating Node) 汇总各个节点的数据，再通过排序选择前1000个文档返回 最后的结果：

![输入图片说明](https://images.gitee.com/uploads/images/2021/1027/093222_b28db590_8076629.png "屏幕截图.png") 

缺点：

这些数据都是在内存中操作的，随着翻页越深入，数据越多，内存消耗越多，这样的效率是非常低的。另外es为了性能，限制了我们分页的深度，es目前支持的最大的 max_result_window = 10000 (可以通过接口调整)

#### scroll

为了满足深度分页的场景，es 提供了 scroll 的方式进行分页读取。原理上是对某次查询生成一个游标 scroll_id（类似数据库中的cursor游标） ， 后续的查询只需要根据这个游标去取数据，直到结果集中返回的 hits 字段为空，就表示遍历结束。scroll_id 的生成可以理解为建立了一个临时的历史快照，在此之后的增删改查等操作不会影响到这个快照的结果，所有文档获取完毕之后，需要手动清理掉 scroll_id。

第一次请求，设置scroll=5m，表示该窗口过期时间为5分钟，分页size大小为10
```
GET /student/student/_search?scroll=5m
{
  "query": {
    "match_all": {}
  },
  "size": 10
}
返回
{
  "_scroll_id" : "DnF1ZXJ5VGhlbkZldGNoBQAAAAAAAC0YFmllUjV1QTIyU25XMHBTck1XNHpFWUEAAAAAAAAtGRZpZVI1dUEyMlNuVzBwU3JNVzR6RVlBAAAAAAAALRsWaWVSNXVBMjJTblcwcFNyTVc0ekVZQQAAAAAAAC0aFmllUjV1QTIyU25XMHBTck1XNHpFWUEAAAAAAAAtHBZpZVI1dUEyMlNuVzBwU3JNVzR6RVlB",
 ...
 }
```
之后每次请求，只要带上scroll_id即可，每次返回size大小的记录数，直到返回结果集中为空，表示没有更多的数据了。

缺点：

scroll 方式是针对本次查询生成一个临时快照，实时变动的数据无法查询到，scroll_id会占用大量的资源（特别是排序的请求）

为了提高性能，scroll分页中还有其他的优化部分，例如：Scroll Scan 结果没有排序，其他与普通scroll类似，但是分页效率要高一些。还有Sliced Scroll 可以并发来进行数据遍历等。

#### Search After

 ES 5 引入的一种分页查询机制，其原理几乎就是和scroll一样，维护一个实时游标，它以上一次查询的最后一条记录为游标，方便对下一页的查询，它是一个无状态的查询，因此每次查询的都是最新的数据，由于它采用记录作为游标，因此SearchAfter要求doc中至少有一条全局唯一变量（每个文档具有一个唯一值的字段应该用作排序规范）

缺点：

1. 由于实时查询，因此在查询期间的变更（增加删除更新文档）可能会导致跨页面的不一值。

2. 至少需要制定一个唯一的不重复字段来排序

3. 它不适用于大幅度跳页查询，或者全量导出，对第N页的跳转查询相当于对es不断重复的执行N次search after，而全量导出则是在短时间内执行大量的重复查询



| 分页方式         | 性能 | 优点                       | 缺点                                                           | 场景                  |
|--------------|----|--------------------------|--------------------------------------------------------------|---------------------|
| from + size  | 低  | 灵活性好，实现简单                | 深度分页问题                                                       | 数据量比较小，能容忍深度分页问题    |
| scroll       | 中  | 解决了深度分页问题                | 无法反应数据的实时性（快照版本）维护成本高，需要维护一个 scroll_id                       | 海量数据的导出需要查询海量结果集的数据 |
| search_after | 高  | 性能最好不存在深度分页问题能够反映数据的实时变更 | 实现复杂，需要有一个全局唯一的字段连续分页的实现会比较复杂，因为每一次查询都需要上次查询的结果，它不适用于大幅度跳页查询 | 海量数据的分页             |

