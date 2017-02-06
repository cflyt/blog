---
title: Mongdb：索引优化和工具
tags: 技术
categories: 未分类
date: 2017-02-06 23:03:46
type:
---


# Mongdb：索引优化和工具

索引能帮助我们快速的找到我们的记录，在数据量庞大的情况下，显得尤其重要。不用索引的查询称为“全表扫描”，对大集合来说，效率非常低。
在Mongodb中增加索引可以：

* 把一个线性查询时间复杂度O(n)变为b树查询时间复杂度为O(lgn);
* 减少在内存中的排序；
* 减少硬盘的访问，减少寻页缺失；
* 索引覆盖，有些查询直接从索引中获得，不需要查记录。

本篇文章的内容如下：
1、索引的相关操作和复合索引；
2、查询索引优化原理和Mongodb查询优化器索引选择机制；
3、索引优化工具。

## 1 索引的相关操作和复合索引

### 1.1 关于索引的基本操作
- `db.collection.ensureIndex(indexed_fields, [options]);` 创建一个索引。
建索引不得不提的是，建索引就是一个容易引起长时间写锁的问题，MongoDB在前台建索引时需要占用一个写锁（而且不会临时放弃），**阻止读和写**，如果集合的数据量很大，建索引通常要花比较长时间，特别容易引起问题。 使用background: true来解决问题，background在建索引中允许读和写操作，减少建索引带来的影响。

- `db.collection.getIndexes();` 查看某个表上的所有索引。
- `db.collection.dropIndex(indexed_fields);` 删除索引。
- `db.collection.find(find_fields).explain();` 查看查询的过程的统计。
- `db.collection.find(fields).hint(index_or_name);` 强制查询使用某个索引。


### 1.2 Mongodb 复合索引
复合索引就是建立在多个字段上的索引，当查询中有多个排序方向或者包含多个键值的条件时，建立复合索引就很有用。
`db.test.ensureIndex({age: 1, role: 1}, {background: true})`
就建立一个在age和role上的索引，先以age升序，如果age相同，就以role升序排序。

**1.隐含索引**
如果有一个由N个键的索引，那么会得到N个键前缀组成的索引，比如有复合索引{a:1, b:1, c:1, d:1},实际上我们同时拥有了{a:1}, {a:1, b:1}, {a:1, b:1, c:1}这三个索引。

**2.排序顺序**
索引以两种顺序存储键，升序 (1) 或降序 (-1)。对于单键索引而言，键的顺序无关紧要因为MongoDB可以以任一顺序遍历整个索引。但是对于 复合索引 而言，索引的顺序可以决定索引是否支持直接排序操作。只有基于多键的排序，索引的顺序才重要。
相互翻转（在每个方向上都乘以-1）的索引是等价的。{age:1, role:1}和{age:-1, role:-1}等价，{age:1, role:-1}和{age:-1, role:1}也是等价的。 但是{age:1, role:1}和{age:1, role:-1}不等价。为什么呢，想想B树的遍历方式，就明白了哈。

`db.test.find().sort({age:1, role:-1)`要求查询结果以age升序，role降序排序，{age:1, role:1}索引就不支持直接排序了，需要进行内存排序{scanAndOrder:true}。


## 2 优化原理
### 2.1 查询索引优化原理

### explain()
explain()能够提供大量与查询相关的信息，对于速度较慢的查询来讲是一个重要的诊断工具之一。

    db.test.find({age: {"$gte": 30 , "$lte": 50}}).explain()
    {
        "cursor" : "BasicCursor",
        "n" : 3,
        "nscannedObjects" : 4,
        "nscanned" : 4,
        "scanAndOrder" : false,
        "indexOnly" : false,
        "millis" : 0
        .....
    }
    cursor:本次所使用的索引，此次没有使用索引，如果使用了索引为BtreeCursor;
    n：此次查询返回的文档数量；
    nscannedObjects：mongodb按照索引指针去查找实际文档次数；
    nscanned：如果有索引，那么这个值就是查找过索引条目的数量，如果没有索引，就是全表扫描，表示检查锅文档的数量。
    scanAndOrder：mongodb是否在内存中对结果进行排序。
    indexOnly：只需要索引就完成了查询，索引覆盖。
    millis：查询所耗费的毫秒数。

因此可以得出**nscanned >= nscannedObjects >= n**。对于简单查询你可能期望3个数字是相等的，scanAndOrder：false，这意味着你做出了MongoDB使用的完美的索引。不过很多时候在索引查找速度和是否在内存排序之间需要做出一个权衡。

### 范围查询
首先建一个在age上的索引{age: 1}，查询30<=age<=50,

    db.test.ensureIndex({age: 1}, {background: true})
    db.test.find({age: {"$gte": 30 , "$lte": 50}}).explain()
    {
        "cursor" : "BtreeCursor age_1",
        "n" : 3,
        "nscannedObjects" : 3,
        "nscanned" : 3,
        "scanAndOrder" : false,
        "indexOnly" : false,
        "millis" : 0,
        ......
    }

建了age字段索引之后，cursor变为"BtreeCursor age_1"，nscanned和nscannedObjects都降为到了3，说明查询使用了索引跳过不符合条件的记录。

### 范围查询+等值查询
如果范围查询基础上，再加上等值查询{role: "female"}，结果如下：

    db.test.find({age: {"$gte": 30 , "$lte": 50}, type: true}).explain()
    {
        "cursor" : "BtreeCursor age_1",
        "n" : 2,
        "nscannedObjects" : 3,
        "nscanned" : 3,
        ....
    }
过滤了role: ”female”，n降为2，但是nscanned和nscannedObjects仍然为3。
创建{age:1, role:1}复合索引又如何？

    db.test.dropIndex({age:1})
    db.test.ensureIndex({age: 1, role: 1}, {background: true})
    db.test.find({age: {"$gte": 30 , "$lte": 50}, role: "female"}).explain()
    {
        "cursor" : "BtreeCursor age_1_role_1",
        "n" : 2,
        "nscannedObjects" : 2,
        "nscanned" : 3,
        .....
    }
nscannedObjects降为2，与n相等，但nscanned还是为3，mongodb做了age30到50之间索引全扫描。从下面的图可以了解查找过程。
![](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2015/01/age-role-index.png)
从上图清晰看到，根据B树的遍历方式，mongodb遍历了30到50之间的索引。

能不能使nscanned = nscannedObjects = n，细心的你回发现，定义索引的顺序有问题，定义{role:1, age:1 }索引，再进行查找，结果如下：

    db.test.dropIndex({age:1, role:1})
    db.test.ensureIndex({ role: 1, age: 1}, {background: true})
    db.test.find({age: {"$gte": 30 , "$lte": 50}, role: "female"}).explain()
    {
        "cursor" : "BtreeCursor role_1_type_1",
        "n" : 2,
        "nscannedObjects" : 2,
        "nscanned" : 2,
        "scanAndOrder" : false,
        ....
    }

nscanned变成了2。
如果数据量有上千万条，缩小nscanned能提升很大的效率。但也要衡量下多值索引带来的内存开销，插入时的开销。


### 范围查询+排序
范围查询加上排序，我们应该怎么来优化呢？
`db.test.find({age: {"$gte": 30 , "$lte": 50}).explain()`
前面的{role:1,age:1}索引放在这里没有作用，Mongodb还是做了全表扫描。
我们来建一个索引{age:1, score:1}，然后查询来分析。

    db.test.ensureIndex({age:1, score:1})
    db.test.find({age: {"$gte": 20 , "$lte": 40}, score: {"$gte":80, "$lte":100}}).sort({score:1}).explain()
    {
        "cursor" : "BtreeCursor age_1_score_1",
        "n" : 2,
        "nscannedObjects" : 2,
        "nscanned" : 2,
        "scanAndOrder" : true,
        ....
    }
nscanned=nscannedObjects=n，似乎达到我们的要求，但是scanAndOrder为true，这就意味着MongoDB会把所有查询出来的结果放进内存，然后进行排序，接着一次性输出结果。这将占用服务器大量的CPU和RAM，而且mongodb对排序还有个限制，就是排序集不能大于32M，否则会出现错误。

创建索引{score:1, age:1 },继续查询得到

    db.test.ensureIndex({score:1, age:1})
    db.test.find({age: {"$gte": 20 , "$lte": 40}, score: {"$gte":80, "$lte":100}}).sort({score:1}).explain()
    {
        "cursor" : "BtreeCursor score_1_age_1",
        "n" : 2,
        "nscannedObjects" : 2,
        "nscanned" : 3,
        "scanAndOrder" : false,
        ....
    }

修改索引后，nscanned虽然为3，但是scanAndOrder为false，免去了排序。这两者之间根据实际情况进行权衡，如果nscanned和n相差特别大，则优先查询，否则优先排序。

### 等值查询+范围查询+排序
如果一个查询包含了等值查询，范围查询，排序，又该如何优化？
`db.test.find({age: {"$gte": 30 , "$lte": 50}, role: "female"}).sort({score:1}).explain()`
对于这个查询，前面已经介绍过，使用{role:1, age:1},将获得nscanned = nscannedObjects = n =2的体验，但是有排序的缺点。
使用索引{role:1, score:1},索引结构如下图，
![](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2015/03/role-score.png")
nscanned = nscannedObjects=3, 不用排序。
以牺牲nscanned的代价解决了scanAndOrder = true的问题；既然nscanned已不可减少，那么我们是否可以减少nscannedObjects？我们向索引中添加age，这样一来Mongo就不用去从每个文件中获取了。
建立索引{role:1, score:1, age:1},然后查询

    db.test.find({age: {"$gte": 30 , "$lte": 50}, role: "female"}).sort({score:1}).hint('role_1_score_1_age_1').explain()
    {
        "cursor" : "BtreeCursor role_1_score_1_age_1",
        "n" : 2,
        "nscannedObjects" : 2,
        "nscanned" : 3,
        "scanAndOrder" : false,
        ....
    }

现在nscannedObjects也降为2了，索引也尽善了，但是多字段的索引需要内存和影响写效率的，增加一个字段如果不能增加nscanned和nscannedObjects之间的差值，应该是得不偿失。

### 2.2 mongodb查询优化器原理
那么优化器如何工作的呢？
首先，它先分析查询，做一个初步的“最佳索引”分析；
其次，假如这个最佳索引不存在， 优化器会为每个可能有效适用于该查询的索引创建查询计划，随后并行运行这个计划，nscanned 值最低的计划胜出。优化器会停止那些长时间运行的计划，将胜出的计划保存下来，以便后续使用。如果有多个最佳索引，mongodb将随机选择一个。
最后优化器还会记住所有类似查询的选择（只到大规模文件变动或者索引上的变动）。
最佳索引是如何选取的呢？

* 最佳索引必须包含查询中所有可以做过滤及需要排序的字段。
* 如果查询包含范围查找或者排序，那么对于选择的索引，其中最后用到的键需能满足该范围查找或者排序。
* 如果存在不同的最佳索引，那么Mongo将随机选择。


表数据是变化的，索引也由可能不是最佳的了，查询优化器什么时候重新评估索引呢？

* 数据表接收1000次写操作后；
* 索引重建；
* 增加或删除索引；
* mongod进程重启；
* 执行explain()操作；

当这些事件发生时，当前的缓存的查询计划会清空，重新评估，以便保证获得最佳的查询计划。

### 2.3 总结优化策略
根据上面的分析，复合索引的顺序应该是

* 等值查询的字段放在前面；
* 接着是需要排序的字段；
* 最后是范围查询的字段。
*
但是有值得权衡的地方，有可能查询访问更多的索引节点，如果排序字段在范围查询字段之前，这两者之间的取舍根据实际情况来定夺。


## 3 Mongodb索引优化工具
这里介绍Mongdb自带的优化器Profiler和第三方工具DEX。

### 3.1 Profiler
类似于MySQL的slow log, Mongodb可以非常方便地记录下所有耗时过长操作，以便于调优。

#### 开启Profiling
Mongodb中Profiler默认是关闭的。有两种方式可以开启和控制Profiling的级别。
第一种在启动mongd实例时，加上-profile=级别；
第二种是在客户端中开启，使用命令db.setProfilingLevel(级别)来配置。
profiling有0，1，2三种级别：
0：不开启记录；
1：记录慢命令（默认是100ms）;
2：记录所有命令；

修改默认慢时间可以在启动mongd实例时添加-slowms=xx或者在shell客户端启用的同时传入参数。
`db.setProfilingLevel(1, 20)`设置的慢时间为20毫秒。
**注意profiling的默认时间的修改是全局起作用的，针对整个mongod实例。**

#### Profiling记录
profiling的记录是在对应数据库下的，名字为system.profile。

    db.system.profile.find().pretty()
    {
     "op" : "query",
     "ns" : "gacc.app_user_duration",
     "query" : {
      "query" : {
       "duration" : {
        "$gt" : 6000,
        "$lt" : 60000
       },
       "tin" : {
        "$gt" : 126414600,
        "$lt" : 995810371
       }
      },
      "orderby" : {
       "timestamp" : -1
      },
      "$explain" : true
     },
     "ntoskip" : 0,
     "numYield" : 0,
     "nreturned" : 1,
     "responseLength" : 770,
     "millis" : 3263,
     "ts" : ISODate("2015-03-11T11:53:41.741Z"),
     "client" : "127.0.0.1",
     "allUsers" : [ ],
     "user" : ""
    }

    ts： 该命令在何时执行
    op: 操作类型
    query: 本命令的详细信息
    responseLength: 返回结果集的大小
    ntoreturn: 本次查询实际返回的结果集
    millis: 该命令执行耗时，以毫秒记

show profile命令可以显示最新的5条操作记录；或者使用字段进行过滤查询。

#### Profiling效率
profiling功能肯定是会影响效率的，但其记录用了更加高效的capped collection。这种collection有一些限制，来增加效率。

1. 固定大小；Capped Collections 必须事先创建，并设置大小：
> db.createCollection("collection", {capped:true, size:100000})
2. Capped Collections 可以insert 和update 操作,不能delete 操作。只能用drop()方法删除整个Collection。

3. 默认基于Insert 的次序排序的。如果查询时没有排序，则总是按照insert 的顺序返回。

4. FIFO。如果超过了Collection 的限定大小，则用FIFO 算法，新记录将替代最先insert的记录.

###3.2 Dex

Dex是一个第三方的开源的Mongodb优化工具，建立在profiler上，但是对索引的分析有一套自己的标准，推荐最佳索引给开发者参考。

#### 工作过程
1. Dex遍历Mongodb的日志或者profile collection；
2. 从输入中Dex的LogParser或者ProfilerParser提取每个查询；
3. 将查询传人到QueryAnalyzer；
4. QueryAnalyzer分析存在的索引，看是否符合自己的优化标准，如果不存在这样的索引，Dex就推荐一个。

#### 优化标准
第一步：解析queryDex会对查询query进行解析，分成下面几大类：

* EQUIV – 普通按数值进行的查询，比如：{a: 1}
* SORT – sort操作，比如： .sort({a: 1})
* RANGE – 范围查询，比如：Specifically: ‘$ne’, ‘$gt’, ‘$lt’, ‘$gte’, ‘$lte’, ‘$in’, ‘$nin’, ‘$all’, ‘$not’
* UNSUPPORTED
    * 组合式查询，比如：$and, $or, $nor，
    * 除了RANGE之外的嵌套查询

第二步：判断当前索引情况有两个标准来找出查询所需的索引。

* Coverage (none, partial, full) - Coverage表示索引覆盖查询条件的情况。none表示完全无索引覆盖。full表示query中的字段都能找到索引。partial表示只有部分字段。
* Order - Order是用于判断索引的顺序是否理想。理想的索引顺序应该是：等值 ○ 排序 ○ 范围.
值得注意的是，对地理位置索引只会进行分析，但是不会提出改进建议。

第三步：通过上面的分析，Dex会生成一个此查询的最佳索引。如果这个索引不存在，并且查询情况不包括上面提到的UNSUPPORTED，那么Dex就会做出相应的索引优化建议。

#### Dex的使用

##### 安装
>pip install dex

##### 启动Dex
1. 以mongodb的log日志为数据源，启动dex。
> dex -f /var/log/mongodb.log mongodb://uutest:xxx@127.0.0.1:27017/gacc

2. 或者以system.profile数据表为分析依据，事先需要开启mongodb的Profiling,推荐开启级别为1的Profiling。
>dex -p mongodb://uutest:xxxx@m127.0.0.1:27017/gacc

##### 过滤器
1. 过滤数据库和数据表
通过 -n 参数可以对数据库和数据表进行过滤，缩小范围，加快分析。
> dex -f my/mongod/data/path/mongodb.log -n "gacc.app_user_duration" mongodb://uutest:xxx@127.0.0.1:27017/gacc
>dex -p -n "*.app_user_duration" mongodb://uutest:xxx@127.0.0.1:27017/gacc
>dex -f my/mongod/data/path/mongodb.log -n "gacc.\*" -n "gacc_stat.\*" mongodb://uutest:xxx@127.0.0.1:27017/gacc

2. 过滤查询时间
 使用-s/--slows加时间，能过滤慢日志，当查询小于这个值就不被Dex分析。
>dex -f my/mongod/data/path/mongodb.log -s 400
>dex -p -n "*.app_user_duration" mongodb://uutest:xxx@127.0.0.1:27017/gacc --slowms 1000

##### 监控模式
Dex提供了实时监控模式，使用-w参数，实时分析新产生的查询，启用监控模式后，对以往旧的记录不分析。
例子：
> dex -w -f my/mongod/data/path/mongodb.log mongodb://myUser:myPass@myHost:12345/myDb


如果既有-w，又有-p， 则必须添加-n进行数据库的过滤，形如[dbname.*],举例如下：

> dex -w -p -n "gacc.*" mongodb://uutest:xxx@127.0.0.1:27017/gacc

而且如果事先没有开启Profiling，dex会自动打开级别为1的Profiling.

##### 分析输出
Dex的分析报告输出有两部分：

1. runStats – 分析日志或profile数据统计
runStats.linesRead – 多少条数母（日志或profile）发送到Dex；
runStats.linesAnalyzed -多少条数目dex成功提取查询，并试图给出建议的数量；
runStats.linesWithRecommendations – 有多少条查询能收益与推荐的的索引；
runStats.logSource – 日志的路径。如果使用 -p/–profile 模式，此项为空.

2. result -：推荐索引的信息包含在results中 ，是一个包含所有查询推荐的list。
 queryMask - 查询模式, 具体的查询条件；
 namespace - 查询的数据库集合collection；
 stats - 针对这个查询做出的统计
    * stats.count - 这个查询出现的次数；
    * stats.avgTimeMillis - 这个查询花费的平均时间
    * stats.totalTimeMillis - 该查询所有的时间消费总和。
 recommendation - 推荐索引的信息
    * recommendation.index - 推荐的索引；
    * recommendation.namespace - 索引所在的数据库集合；
    * recommendation.shellCommand - 推荐的建索引的命令。

 从Dex的统计信息和索引推荐信息，我们可以知道我们的索引存在的问题，还可以参考它分析出来的索引推荐，帮助我们更好的选择索引。


 ## 4 建议
什么时候需要优化索引呢，个人认为有以下几种：
* 如果 nscanned 远大于 nreturned；
* 索引覆盖，如果索引就能包含返回值；
* 执行 update 操作时同样检查一下 nscanned，并使用索引减少文档扫描数量；
* 使用 db.eval() 在服务端执行某些统计操作；
* 减少返回文档数量，使用 skip & limit 分页；
* 数据库文档数量很大，但是只查询少数的的文档。

如果很少读，那么尽量不要添加索引，因为索引越多，写操作会越慢。如果读量很大，那么创建索引还是比较划算的。如果写查询量或者update量过大的话，多加索引是会有好处的。