---
title: MongoDB：锁和并发控制
tags: 技术
categories: 未分类
date: 2017-02-06 23:10:04
type:
---

# MongoDB：锁和并发控制

本文主要介绍两部分内容，第一部分是MongoDB的锁，第二部分是MongoDB的并发控制。这都跟MongoDB性能和效率相关的，有助于在使用MongoDB过程中，知道哪些操作会带来怎样效率的影响。第二部分从业务层面出发介绍并发控制的方法,基本上都是开发中碰到的常见场景。

## MongoDB锁
###锁类型
Mongodb使用的是read-write读写锁，允许多个读者并发的加共享锁访问同一资源，而同一时间只允许一个写者加排它锁改变资源，此时不允许其它的读和写。
在mongodb中，还使用了意向锁（IS、IX）提高加锁性能。为了提高吞吐率，在等待排队的锁中，排在最前面的如果是共享锁，会一次把队列中所有的共享锁都加上，释放之后就处理排他锁。而不是当前是共享锁，后面一来共享锁请求都加上去，后面的请求是要排队的，避免写锁的饥饿。

### 锁的粒度

mongodb普遍使用全局锁、数据库锁。还允许自己使用不同存储引擎，实现更细粒度的锁控制。比如MMAPv1存储引擎支持表锁，WiredTiger存储引擎，支持文档级别的锁。

在 2.2 版本以前，mongodb只有**全局锁**；也就是对整个mongodb实例加锁，这个粒度是很大的。

从 2.2 版本开始，大部分读写操作只锁一个库，库级别锁相对之前版本，这个粒度已经下降。对于一些涉及到多个数据库操作，还需要加全局锁。目前MongoDB常见的版本是2.6、2.7，锁粒度只到collection。

从 3.0 开始，如果使用MMAPv1存储引擎，最细粒度支持 **collection级别的锁** ，对于一个数据库里面的collection，对一个collection的加锁操作，不影响其他collection的操作。



### 查看锁的状态

* [db.serverStatus()](http://docs.mongodb.org/v2.6/reference/command/serverStatus/)
* [db.currentOp()](http://docs.mongodb.org/v2.6/reference/method/db.currentOp/#db.currentOp)
* mongotop
* mongostat

### 产生加锁的操作

操作 |    锁类型
--------------- | ----------
query | 读锁
cursor get more | 读锁
insert |    写锁
remove |    写锁
update |    写锁
Map-reduce|读锁和写锁，除非Map-reduce操作被声明为非原子性的。部分map-reduce任务能够并发的执行。
createIndex|    创建索引操作默认在前台执行，会长时间锁住数据库。
db.eval()|  写锁 db.eval()会加全局写锁，会阻塞其他的读写操作。加上参数nolock: true，表示不加锁。
eval|   写锁. 同db.eval()
aggregate()|    读锁

###读和写操作让出锁
在需要长时间读和非原子写的操作，在一些条件下会让出所占的锁。比如update更新多个文档时候，就有可能在中间让出锁。Mongodb使用一种自适应的算法基于预测磁盘访问（比如缺页中断）方式决定是否让出锁。Monodb在读之前预测所需要的数据是否在内存中，如果预测不在内存中则让出锁，等数据加载后，重新请求上锁来完成操作。在2.6版本以后，对于索引，就算不在内存中，也不让出锁。

### 管理命令数据库锁
在运行下面的管理命令，需要长时间占锁进行操作：

* db.collection.createIndex(), 如果没有设置background:true参数,长时间锁住数据库。
* reIndex；
* compact；
* db.repairDatabase(),
* db.createCollection(), 比如要创建很大的固定空间capped collection时候；
* db.collection.validate(), 返回磁盘信息；
* db.copyDatabase(). 会锁住多个数据库。

对于上面的操作如果有副本集，建议将其中的一个副本下线后进行操作，这样就不会影响上线服务。


下面的管理命令只占短时间锁数据库：

* db.collection.dropIndex(),
* db.getLastError(),
* db.isMaster(),
* rs.status()，
* db.serverStatus(),
* db.auth(),
* db.addUser()

### 锁多个数据库的全局锁
* db.copyDatabase() 全局锁，锁整个monogdb的实例，不允许读写。
* db.repairDatabase() 全局写锁。
* Journaling，如果开启了Journaling功能，在记录日志时会很短时间锁住所有的数据库。所有的数据库共享一个journaling。
对副本集主节点的所有写操作不仅会锁住目标库，也会锁住local库很小一段时间。通过Local库上的锁， mongod进程可以往主节点的oplog 表写数据，以及一些认证操作，不过这些都只占整个写操作时间的很少一部分。

### 分片上锁和并发

分片通过在多个mongod实例部署分布式集合来提高并发的性能，允许分片服务器(比如mongos进程)访问多个mongd实例执行任意数量的并发操作。

mongod实例之间是相互独立的，并且使用的是MongoDB多读单写锁。mongod实例上的操作不会阻塞其它实例。


### 副本集主节点上的锁和并发

在副本集中，当往主节点上某个表写入时，MongoDB也会对主节点的oplog进行写入，这是local库上一个特殊的表。因此，MongoDB会锁住目标表的库和local库。mongod进程会锁住这两个库来保证数据一致性，让这两个库的写操作要么全执行要么全不执行。


### 副本集次级节点上的锁和并发

在副本集中，次级节点会分批的收集oplog信息，这些批次的回写是并行执行的。当在进行写操作时，次级节点上的数据是不可读的，回写操作会按照oplog记录的顺序执行。

MongoDB允许副本集次级节点上的几个写操作并行执行，分两个阶段：
第一个阶段，通过一个读锁，mongod确保所有与写操作有关数据都存在于内存中。在这个阶段，其它客户端可以在这个节点上做查询操作。
第二阶段Mongodb会使用一个线程池使用写锁允许所有的写操作相互协调的成批写入。

## MongoDB并发控制方法
Mongodb不支持事务，这是Mongodb的缺陷，要达到业务方面实现并发，保证数据的一致性，要使用一些并发的控制方法。

### 原子操作
mongodb对单个文档的写都是原子操作，就算修改一个文档里面的多个内嵌文档也算修改一个文档，也是原子操作。
当一次写操作中要修改多个文档，每个文档的写操作是原子的，但整个操作不是原子的，期间会让出锁，允许其他线程操作。
使用update和findAndModify方法，如果都只对一个文档进行操作，那么都是原子的。区别是返回的内容不一样，findAndModify还可以返回当前改完之后的文档，这个在很多场景很有用，比如实现一个自动增长的id。如果用update，然后findOne就不能保证返回的结果是刚刚更新后的记录，可能别的线程又修改了。

### $isolated
Mongodb中有个$isolated操作符，可以隔离其他的线程。 使用$isolate操作符时，可以阻止一次写多个文档中让出锁，直到操作完成或者出错。
但是$isolate也无法保证多个文档修改的原子性(all-or-nothing），$isolate操作如果在写过程中出错，是不会回滚的，也就是存在只修改部分文档的情况。可以这么说$isolated操作符只提供隔离性，但不保证原子性。
**操作符$isolated不能在分片上工作。**
下面是使用$isolated操作符进行update的例子：

    db.test.update(
        { age : {$gt:30} , $isolated : 1 },
        { $inc : { score : 1 } },
        { multi: true }
    )

当执行此操作时，其他线程就阻塞了，直到把所有操作更新完。这个操作会锁数据库比较长的时间，影响并发的效率，注意进行权衡。

#### 使用唯一性索引
当你想要创建某个字段唯一的文档时候，在多线程环境下，使用先查询为空再插入的方法，仍会导致相同的字段记录产生。解决的方法是在这个字段上建一个唯一的索引，在插入相同的字段时，会raise duplicate index key error。

    db.members.createIndex( { "name": 1 }, { unique: true } )

上面创建了一个name字段的唯一索引。
然后使用操作insert或者update方法增加和修改文档，捕捉raise的duplicate index key错误，判断是否操作成功。比如使用update操作：

    db.people.update(
       { name: "cf" },
       {
          name: "cf",
          age: 18,
          score: 100,
       },
       { upsert: true }
    )

用upsert参数表明如果不存在则进行插入操作，这样做能保证name字段的值是唯一的，即便在多线程环境中。

### 事务
如果文档的内容有两个field，A和B，现在想根据B的值进行修改A的值，比如（A=B+N），也就是需要知道field B的内容，才能进行A修改。这时候使用update和findAndiModify就做不到了。
有两种方法。
一种是两阶段提交事务语义(two-phase commit)方法，另外一种是update if current。

#### update if current

update if current是用手动的方式保证原子性，类似于CPU中常见的同步原语:比较和交换cas（Compare & Swap）操作，原理如下：
1、获取需要更新的记录，并记录A字段的值，oldvalue；
2、本地修改好要更新的字段A；
3、使用update操作修改A字段值，但是查询条件必须是A字段的值==oldvalue。
下面举例说明：

    var myDocument = db.test.findOne( { name: 'cf' } );

    if (myDocument) {

      var oldValue = myDocument.score;
      var age = myDocument.age

      if (myDocument.score < 10) {
              myDocument.score *= 5;
      } else {
          myDocument.score = age * 1.5;
      }

      var results = db.test.update(
         {
           name: myDocument.name,
           score: oldValue
         },
         {
           $set: { score: myDocument.score }
         }
      );

      if ( results.hasWriteError() ) {
          print("unexpected error updating document: " + tojson( results ));
      } else if ( results.nMatched == 0 ) {
          print("No update: no matching document for { name: " + myDocument.name + ", score: " + oldValue + " }")
      }

上面的例子，score的值修改，跟score本身、age值相关，这就需要先获取score、age的值，再进行修改。
你可以发现，这需要自己手动去检查有没有修改成功，如果不成功还需自己去控制再次尝试更新。


#### 两阶段提交事务
第一个阶段尝试进行提交，第二个阶段正式提交。这样，即使更新数据时发生故障，我们也能知道数据都处于什么状态，总是能够把数据恢复到更新之前的状态。原理是使用一个表来记录事务transaction，保存可能的状态inital、pending、applied、done、canceling、canceled；要进行更新的collection的记录中增加一个类型为列表pendingTransactions字段，用来绑定正在执行的transaction id。通过修改transaction里面记录的状态，确保事务执行的阶段，这样在发生故障时就知道transaction处于哪个阶段，从而resume或者rollback。
具体例子可以看文档：[Two Phase Commits
](http://docs.mongodb.org/manual/tutorial/perform-two-phase-commits/)
两阶段提交方法达到事务一致性，我个人觉的还是比较麻烦，这也是mongodb的缺陷所在。



参考文档：
<http://askasya.com/post/findandmodify>
<http://docs.mongodb.org/v2.6/faq/concurrency/>
<http://docs.mongodb.org/manual/core/write-operations-atomicity/>