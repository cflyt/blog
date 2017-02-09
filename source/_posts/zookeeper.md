---
title: Zookeeper 分布式应用协调服务
tags: 技术
categories: 未分类
date: 2017-02-09 09:10:23
type:
---


## Zookeeper 分布式应用协调服务

本想先介绍Kafka这个开源分布式消息中间件的，这框架使用了Zookeeper，Zookeeper的出现就是为分布式应用开发而设计的，很有实际应用，值得介绍一番。
为了初步了解Zookeeper能干什么，我先抛出一个实际需要解决的场景，就拿UU来举个例子。

### 解决场景

  UU项目是LINK|APP的架构，客户端通过访问CDN上面的记录LINK地址静态资源文件获取LINK的IP地址，得到LINK地址后发起访问。LINK|APP有两台备机，当有服务宕机时，需要手动修改静态资源文件，去掉宕机的机器地址，加入备机机器的地址，然后push到CDN，完成切换。

这里面很容易发现几个问题：

* LINK|APP是1对1的，LINK和APP任何一个服务崩溃，尽管另一个服务正常运作也变的不可用；
* 从发现LINK|APP服务宕机，到备机上线，再到push到CDN，这几步都是手动的，不够自动化，花费的时间很长；

现在我想要达到这样的效果：

* LINK和APP是多对多的，LINK能动态知道哪些APP可用，也就是说LINK以服务发现的方式使用APP；
* 备机能检测到LINK服务的宕机，并自动把自己切换成线上可用；
* CDN客户端能自动检测LINK服务地址的变动，并动态修改资源文件，然后push到CDN服务中；

Zookeeper可以帮我们做到这些，如何做到的呢？

### Zookeeper是什么
在进程内进行协调我们可以使用语言，平台，操作系统等为我们提供的机制。但是如果我们在一个分布式环境中呢？也就是我们的程序运行在不同的机器上，这些机器可能位于同一个机架，同一个机房又或不同的数据中心。分布式协调比同一个进程里的协调复杂得多，复杂的原因是网络是不可靠的，容易陷入一些诸如竞争选择条件或者死锁的陷阱中。在这样的环境中，我们要实现协调该怎么办？这就是分布式协调服务要干的事情。

可能大家知道Google的chubby，chubby是实现Google的一个分布式锁的实现，运用到了paxos算法解决的一个分布式事务管理的系统。chubby是没有开源的，对应的在开源界我们就有了ZooKeeper，Zookeeper就是雅虎模仿强大的Google chubby实现的一套分布式锁管理系统。同时，Zookeeper分布式服务框架是Apache Hadoop的一个子项目，它是一个针对大型分布式系统的可靠协调系统，它主要是用来解决分布式应用中经常遇到的一些数据管理问题，可以高可靠的维护元数据，保证一致性。可以实现的功能包括：配置管理、命名服务、分布式同步、锁服务等。ZooKeeper的设计目标就是封装好复杂易出错的关键服务，将简单易用的接口和性能高效、功能稳定的系统提供给用户。


### Zookeeper服务架构
在使用Zookeeper中，我们都是通过Zookeeper Client API与Zookeeper Server进行交互，是一种C/S架构，下图是Zookeeper服务的典型架构：

![Zookeeper系统架构](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2015/07/Zookeeper-framework1.png)

Zookeeper服务可以以两种方式执行，单机模式和集群模式。 单机模式下，仅有一个Zookeeper服务器，Zookeeper状态不会被复制。 集群模式会使用Quorum机制，是一种分布式系统中常用的，用来保证数据冗余和最终一致性的投票算法。 集群模式下，一组Zookeeper服务器，状态会被复制，并一同为客户端提供服务。

#### 集群模式
集群模式有三种角色：leader、follower、客户端。
集群使用 **Zab** 、**Paxos** 分布式一致性算法进行数据的同步，leader也是这个算法中产生的，一个leader确定好后，其他自动变成follower。
客户端需要指定连接服务器，可以配置多个服务器地址，客户端会尝试连接其中一个服务器地址；客户端的请求先到达所连的服务器，这个服务器有可能是follower或者leader，如果连的是follower，而且更新数据，follower把数据状态复制给leader，leader与其他follower进行同步。
集群中只容忍小于N/2个Server崩溃，超过半数机器不可用，整个集群也变的不可用，集群建议用奇数个Server组成。
**客户端所在的环境就是分布式环境，Zookeeper协调这些客户端，提供可靠的，一致性的服务。**


### Zookeeper数据模型
Zookeeper提供基于类似于文件系统的目录节点树方式的数据存储，但是Zookeeper并不是用来专门存储数据的，它的作用主要是用来维护和监控你存储的数据的状态变化。**通过监控这些数据状态的变化，从而可以达到基于数据的集群管理。**
下面我们来看一下这个数据结构：

![zookeeper-model](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2015/07/Zookeeper-model.gif)

如上图所示，ZooKeeper数据模型的结构与Unix文件系统很类似，整体上可以看作是一棵树，每个目录子项称做一个ZNode。我们能够自由地增加、删除ZNode，在一个ZNode下增加、删除子ZNode；唯一的不同在于ZNode是可以存储数据的。Zookeeper有四种类型的ZNode：

* PERSISTENT-持久化目录节点：客户端与Zookeeper断开连接后，该节点依旧存在；
* PERSISTENT_SEQUENTIAL-持久化顺序编号目录节点：客户端与Zookeeper断开连接后，该节点依旧存在，只是Zookeeper会给该节点名称进行顺序编号；
* EPHEMERAL-临时目录节点：客户端与Zookeeper断开连接后，该节点被删除；
* EPHEMERAL_SEQUENTIAL-临时顺序编号目录节点：客户端与Zookeeper断开连接后，该节点被删除，只是 Zookeeper会给该节点名称进行顺序编号。

Zookeeper这种数据结构有如下这些特点：

* 每个子目录项如NameService都被称作为znode，这个ZNode是被它所在的路径唯一标识的，如Server1这个 ZNode的标识为/NameService/Server1；
* ZNode可以有子节点目录，并且每个ZNode可以存储数据，注意 **EPHEMERAL** 类型的目录节点不能有子节点目录；
* ZNode是有版本的，每个ZNode中存储的数据可以有多个版本，也就是一个访问路径中可以存储多份数据，比如某一个路径下存有多个数据版本，那么查询这个路径下的数据就需要带上版本；
* ZNode 可以是临时节点，一旦创建这个ZNode的客户端与服务器失去联系，这个ZNode也将自动删除，Zookeeper的客户端和服务器通信采用长连接方式，每个客户端和服务器通过心跳来保持连接，这个连接状态称为session，如果ZNode是临时节点，这个session失效，ZNode也就删除了；
* ZNode 的目录名可以自动编号，如App1已经存在，再创建的话，将会自动命名为App2；
* ZNode可以被监控，包括这个目录节点中存储的数据的修改，子节点目录的变化等，一旦变化可以通知设置监控的客户端，**这个功能是Zookeeper对于应用最重要的特性，** 通过这个特性可以实现的功能包括配置的集中管理，集群管理，分布式锁等等。


Zookeeper为集群提供了一个共享存储库，客户端可以从这里集中读写共享的信息，避免了每个客户端机器的共享操作编程，减轻了分布式系统的开发难度。
Zookeeper的设计采用的是 **观察者的设计模式** ，Zookeeper主要是负责存储和管理大家关心的数据，然后接受观察者的注册，一旦这些数据的状态发生变化，Zookeeper 就将负责通知已经在 Zookeeper 上注册的那些观察者做出相应的反应，从而达到管理功能。


### Zookeeper API
Zookeeper本身并没有提供一些原语操作，如分布式锁，而是，暴露一些 **类似文件系统的API**：

    * create(path, data, flags): 创建一个ZNode, path是其路径，data是要存储在该ZNode上的数据，flags常用的有: PERSISTEN, PERSISTENT_SEQUENTAIL, EPHEMERAL, EPHEMERAL_SEQUENTAIL；
    * delete(path, version): 删除一个ZNode，可以通过version删除指定的版本, 如果version是-1的话，表示删除所有的版本；
    * exists(path, watch): 判断指定ZNode是否存在，并设置是否Watch这个ZNode。这里如果要设置Watcher的话，Watcher是在创建ZooKeeper实例时指定的，如果要设置特定的Watcher的话，可以调用另一个重载版本的exists(path, watcher)。以下几个带watch参数的API也都类似；
    * getData(path, watch): 读取指定ZNode上的数据，并设置是否watch这个ZNode；
    * setData(path, watch): 更新指定ZNode的数据，并设置是否Watch这个ZNode；
    * getChildren(path, watch): 获取指定ZNode的所有子ZNode的名字，并设置是否Watch这个ZNode；
    * sync(path): 把所有在sync之前的更新操作都进行同步，达到每个请求都在半数以上的ZooKeeper Server上生效，path参数目前没有用；
    * setAcl(path, acl): 设置指定ZNode的Acl信息；
    * getAcl(path): 获取指定ZNode的Acl信息；

仅依靠这些API就已能实现配置管理、命名服务、集群管理、分布式锁等场景。


### Zookeeper应用
Zookeeper应用很广泛，在一些著名的开源分布式框架中使用了Zookeeper的的协调服务。

1. Apache HBase。HBase中，Zookeeper用于选举集群Master，跟踪可用的Server，和保存集群元数据。
2. Apache Kafka。Kafka中，Zookeeper用于崩溃检测，实现Topic发现，和维护Topic的生产和消费状态。
3. Apache Solr。Solr中，Zookeeper用于存储集群的元数据信息及协调元数据的更新。
4. Yahoo!Fetching Server。Fetching Service中，Zookeeper用于Master选举，崩溃检测，元数据保存。
5. Facebook Messages。Messages中，Zookeeper用于实现分片和故障迁移的控制器，和服务发现。
..............

下面简单介绍几种场景，加深对Zookeeper实际应用的理解。

### 命名服务
分布式应用中，通常需要一套完备的命令机制，既能产生唯一的标识，又方便人识别和记忆。 只要在ZooKeeper的文件系统里创建一个ZNode，就能获得唯一的path。这个path就能作为一个名称。另外ZNode上还可以存储少量数据，通过使用命名服务，客户端应用能够根据指定的名字来获取资源服务的地址，提供者等信息。被命名的实体通常可以是集群中的机器，提供的服务地址，进程对象等等。

### 配置管理
具体示例：
a）应用中用到的一些配置信息集中管理，在应用启动的时候主动来获取一次，并且在节点上注册一个Watcher，以后每次配置有更新，实时通知到应用，获取最新配置信息；
b）业务逻辑中需要用到的一些全局变量，比如一些消息中间件的消息队列通常有个offset，这个offset存放在zk上，这样集群中每个发送者都能知道当前的发送进度。

### 分布式锁

我们可以利用临时节点来实现，多个进程都尝试创键临时节点/lock， 但最终只会有一个进程P能创建成功，而其他没能创建成功的进程，可以在节点/lock上Watch(相当于等待锁释放)， 一旦进程P处理完事务，断开连接，节点/lock被自动删除，其他进程将得到通知，进而继续创建节点/lock，以争得锁资源。 (这里使用临时节点，是为了防止获得锁的进程突然崩溃而没有释放锁，导致死锁发生)。


### 服务发现
回到文章开头的例子，我们来介绍下如何实现。
①创建/links节点；
②link服务起来的时候，注册自己的服务，即在/links节点下面创建临时节点，注意是临时节点，路径为/links/linkn；
③同样，创建/apps节点；
④app服务起来的时候，也注册自己的服务，在/apps节点下面创建临时节点，路径为apps/appn；
⑤每个link服务通过读取/apps节点下面的所有子节点得到app服务，实现一个link对应多个app；同时设置监听/apps节点，一旦/apps下面的子节点增加、删除，link都能迅速的同步到，从而选择有效的app节点；
⑥link备机设置监听/links节点，一旦/links目录下子节点数量发生变化，如有link崩溃，则把自己注册到/links目录下面，切换成线上状态；
⑦CDN客户端设置监听/links节点，一旦/links目录发生变化，则重新读取link的地址，然后push到CDN。

![find-server](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2015/07/server-find.png)

上面左图就是Zookeeper服务维护的数据，右图是整个场景实现的架构图。

#### 屏障
#### 集群管理
#### leader选举
#### 分布式队列


这些都是 Zookeeper 的基本功能，最重要的是 Zookeeper 提供了一套很好的分布式集群管理的机制，就是它这种 基于层次型的目录树的数据结构，并对树中的节点进行有效管理，从而可以设计出多种多样的分布式的数据管理模型，而不仅仅局限于上面提到的几个常用应用场景。

参考文献：
<http://zookeeper.apache.org/doc/current/>
<https://www.ibm.com/developerworks/cn/opensource/os-cn-zookeeper/>
<http://coolxing.iteye.com/blog/1871520>