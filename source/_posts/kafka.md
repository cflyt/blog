---
title: Kafka消息中间件介绍
tags: 技术
categories: 未分类
date: 2017-02-07 10:26:30
type:
---

# Kafka消息中间件介绍

Kafka是由LinkedIn开发的，分区的、多副本的、多订阅者，基于zookeeper协调的分布式日志系统，作为消息的中间件，它以`水平拓展`、`高可用性`和`高吞吐量`著称，在同类的消息中间件中独树一帜。目前越来越多的开源分布式处理系统如ClouderaApache Storm、Spark都支持与Kafka集成。

#### Kafka设计特点：
1. 以时间复杂度为O(1)的方式提供消息持久化能力，即使对TB级以上数据也能保证常数时间的访问性能；
2. 高吞吐率。即使在非常廉价的商用机器上也能做到单机支持每秒100K条消息的传输；
3. 支持Kafka Server间的消息分区，及分布式消费，同时保证每个partition内的消息顺序传输；
4. 同时支持离线数据处理和实时数据处理；
5. Scale out：支持`在线`水平扩展。

Kafka优秀的设计理念和突出的性能，被称为下一代分布式消息系统，淘宝消息中间件团队完全借鉴了Kafka的设计，用java实现了meta消息中间件，并广泛用于淘宝的业务中。


### Kafka架构
下图是Kafka的主要架构图：
![](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2015/08/Snip20150817_1.png)

组成部件：

* broker：Kafka消息中间件处理结点，一个Kafka节点就是一个broker，多个broker可以组成一个Kafka集群。
* topic：每个消息都属于一个topic，可以看作一个queue，Kafka集群能够同时负责多个topic的分发；
* partition：topic物理上的分组，一个topic可以分为多个partition，每个partition是一个有序的队列；
* producer：负责发布消息到Kafka broker；
* consumer ：负责从Broker pull消息；
* consumer group：每个consumer属于一个特定的consumer group（可为每个consumer指定group name，若不指定group name则属于默认的group）；
* zookeeper：协调服务，用于管理broker服务、分区信息、消费者服务、消费者消费状态等。

下面分别介绍各个部件。

### Kafka存储
#### *文件结构*
再介绍两个名词：

* segment：partition物理上由多个segment组成；
* offset：每个partition都由一系列有序的、不可变的消息组成，这些消息被连续的追加到partition中。partition中的每个消息都有一个连续的序列号叫做offset，用于partition中唯一标识的这条消息。

一个topic可以认为是一类消息，在Kafka文件存储中，同一个topic下有多个不同partition，每个partition为一个目录，partiton命名规则为topic名称+有序序号，第一个partiton序号从0开始，序号最大值为partitions数量减1。

每个partion(目录)相当于一个大文件被平均分配到多个大小相等segment(段)数据文件中。但每个段segment file消息数量不一定相等，这种特性方便old segment file快速被删除，可以在配置中指定删除策略。

segment file组成：由2大部分组成，分别为index file和data file，此2个文件一一对应，成对出现，后缀".index"和“.log”分别表示为segment索引文件、数据文件。

segment文件命名规则：partion全局的第一个segment从0开始，后续每个segment文件名为上一个segment文件最后一条消息的offset值。数值最大为64位long大小，19位数字字符长度，没有数字用0填充。

下面是名字为topicName，分区为2的文件存储结构。
![](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2015/08/Snip20150817_2.png )

topicName-0和topicName-1这两个目录为两个分区，每个分区下面都有两个segment文件组成，文件以上个文件最后一条信息的offset命名。

#### *顺序读写*

Kafka的开发者们认为不需要在内存里缓存什么数据，操作系统的文件缓存已经足够完善和强大，只要你不搞随机写，顺序读写的性能是非常高效的（经验证，顺序写磁盘效率比随机写内存还要高，这是Kafka高吞吐率的一个很重要的保证）。Kafka的数据只会顺序append，数据累积到一定程度或者超过一定时间会被删除。

![](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2015/08/Snip20150817_4.png )
上图是写消息的示意图，每个分区独立append消息。

Kafka读的时候，需要根据offset先找到对应的消息所在的segment，即.index和.log文件，通过索引文件，快速定位到.log文件的文件，然后顺序读到offset消息即可。Kafka读取特定消息的时间复杂度为O(1)，即与文件大小无关，保证了效率。


### producer
producer发送消息到broker时，会根据paritition机制选择将其存储到哪一个partition。如果partition机制设置合理，所有消息可以均匀分布到不同的partition里，这样就实现了负载均衡。如果一个topic对应一个文件，那这个文件所在的机器I/O将会成为这个topic的性能瓶颈，而有了partition后，不同的消息可以并行写入不同broker的不同partition里，极大的提高了吞吐率。可以在$Kafka_HOME/config/server.properties中通过配置项num.partitions来指定新建topic的默认partition数量，也可在创建topic时通过参数指定，同时也可以在topic创建之后通过Kafka提供的工具修改。

在发送一条消息时，可以指定这条消息的key，Producer根据这个key和partition机制来判断应该将这条消息发送到哪个Parition。Paritition机制可以通过指定Producer的paritition. class这一参数来指定，该class必须实现Kafka.producer.partitioner接口。

### consumer
####push和pull
在JMS实现中，topic模型基于push模式，即broker将消息推送给consumer端。不过在Kafka中，采用了pull 模式，即consumer在和broker建立连接之后，主动去pull(或者说fetch)消息； 事实上，push模式和pull模式各有优劣。

* `push模式`很难适应消费速率不同的消费者，因为消息发送速率是由broker决定的。push模式的目标是尽可能以最快速度传递消息，但是这样很容易造成consumer来不及处理消息。
* `pull模式`有它的优点，consumer端可以根据自己的消费能力适时的去fetch消息并处理，且可以控制消息消费的进度(offset)；

Kafka另一个独特的地方是将消费者消费进度保存在客户端而不是服务器，这样服务器就不用记录消息的投递过程，每个客户端都自己知道自己下一次应该从什么地方什么位置读取消息。Kafka还强调减少数据的序列化和拷贝开销，它会将一些消息组织成Message Set做批量存储和发送，并且客户端在pull数据的时候，尽量以zero-copy的方式传输，利用sendfile（对应java里的 FileChannel.transferTo/transferFrom）这样的高级IO函数来减少拷贝开销。

#### consumer group
同一分区的消费是有序的，但跨分区消息的消费不保证顺序性。Kafka为消费者提供了GroupID功能，拥有同样GroupID的消费者，消费不同的分区。上文提过，JMS规范提供了点对点和发布/订阅两种消费模式。如果每个消费者拥有同样的GroupID，那么这个主题就是被当做点对点模式消费；如果拥有不同的GroupID，这个主题就是被当做发布订阅者模式消费。
![](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2015/08/Snip20150817_7.png )
如上图所示，消费者C1和C2在同一个消费组，C1消费P0和P3分区，C2消费P1和P2分区，C1和C2不会相互消费相同的分区。但是C3~C6属于两外一个消费组。那么，C3也可消费P0。如果只有Group A，那么可认为是点对点消费模式。如果有Group A和Group B同时订阅P0，就是发布订阅模式。

### 备份 和 主从分区
Kafka从0.8开始提供partition级别的replication，将分区复制多份，放在不同节点，这就实现了数据的高可用。replication的数量可在$Kafka_HOME/config/server.properties中配置。

    default.replication.factor = 1 默认值为1，不备份

Kafka将每个partition数据复制到多个server节点上，任何一个partition有一个leader和多个follower；leader处理所有的read-write请求，follower需要和leader保持同步。follower和consumer一样，消费消息并保存在本地日志中；leader负责跟踪所有的follower状态，如果follower"落后"太多或者失效，leader将会把它从replicas同步列表中删除。当所有的follower都将一条消息保存成功，此消息才被认为是"committed"，那么此时consumer才能消费它。即使只有一个replicas实例存活，仍然可以保证消息的正常发送和接收，只要zookeeper集群存活即可。(不同于其他分布式存储，比如hbase需要"多数"存活才行)
    当leader失效时，需在followers中选取出新的leader，可能此时follower落后于leader，因此需要选择一个"up-to-date"的follower。在这种模式下，对于f+1个Replica，一个partition能在保证不丢失已经commit的消息的前提下容忍f个Replica的失败。
    作为leader的server节点承载了全部的请求压力，有多少个partitions就意味着有多少个"leader"，Kafka会将"leader"均衡的分散在每个实例上，来确保整体的性能稳定.

回到Kafka架构图中，Kafka集群有两个服务节点，broker1和broker2，topicA有4个分区，分区0、1、2、3，分区0和分区1的leader在broker1上，备份在broker2上，分区2和分区3的leader在broker2上，备份在broker1，producer和consumer的读写都是从leader分区上操作。

### Message Delivery Semantics

对于JMS实现，消息传输担保非常直接：有且只有一次(exactly once)。在Kafka中稍有不同,对于consumer而言：
1) at most once: 消息可能会丢，但绝不会重复传输；
2) at least once: 消息至少发送一次，如果消息未能接受成功，可能会重发，直到接收成功；
3) exactly once: 消息只会发送一次；

* *at most once* : 消费者fetch消息，然后保存offset，然后处理消息；当client保存offset之后，但是在消息处理过程中consumer进程失效(crash)，导致部分消息未能继续处理。那么此后可能其他consumer会接管，但是因为offset已经提前保存，那么新的consumer将不能fetch到offset之前的消息(尽管它们尚没有被处理)，这就是"at most once"。
* *at least once* : 消费者fetch消息，然后处理消息，然后保存offset。如果消息处理成功之后,但是在保存offset阶段zookeeper异常或者consumer失效，导致保存offset操作未能执行成功，这就导致接下来再次fetch时可能获得上次已经处理过的消息，这就是"at least once"。
* *exactly once* : Kafka中并没有严格的去实现(需要基于2阶段提交事务，协调offset和实际操作的输出)。

因为"消息消费"和"保存offset"这两个操作的先后时机不同，导致了上述3种情况。Kafka默认保证At least once，并且允许通过设置producer异步提交来实现At most once。而Exactly once要求与目标存储系统协作，Kafka提供的offset可以使用这种方式非常直接非常容易。

### Zookeeper
Kafka使用zookeeper来发现服务，负载均衡，leader选举等。

1. producer端使用zookeeper用来"发现"broker列表，以及和topic下每个partition leader建立socket连接并发送消息；
2. broker端使用zookeeper用来注册broker信息，监测partition leader存活性；
3. consumer端使用zookeeper用来注册consumer信息，其中包括consumer消费的partition列表等，同时也用来发现broker列表，并和partition leader建立socket连接，并获取消息；
4. 维护offset信息和topic分区信息；

### Kafka性能

LinkedIn团队的性能测试，对比Kafka与Apache ActiveMQ V5.4和RabbitMQ V2.4的性能。ActiveMQ使用默认的消息持久化库Kahadb。LinkedIn在两台Linux机器上实验，每台机器的配置为8核2GHz、16GB内存、6个磁盘使用RAID10。两台机器通过1GB网络连接。一台机器作为代理，另一台作为生产者或者消费者。
在 *生产者测试* 中，对每个系统，运行一个生产者，总共发布1000万条消息，每条消息200字节。 Kafka生产者以1和50批量方式发送消息。ActiveMQ和RabbitMQ似乎没有简单的办法来批量发送消息，LinkedIn每次发送一条消息测试。
在 *消费者测试* 中，LinkedIn使用一个消费者获取总共1000万条消息。LinkedIn让所有系统每次拉请求都预获取大约相同数量的数据，最多1000条消息或者200KB。对ActiveMQ和RabbitMQ，LinkedIn设置消费者确认方式为自动确认。
测试结果如下：
![](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2015/08/Snip20150817_10.png )

从上图可以看到，Kafka如果是批量发送，速率可以到达将近50万条每秒，单条发送也比另外的消息队列高；
消费速率可以达到20万条每秒，远超出其他两个消息队列。

### Kafka适合场景

* *网站活动跟踪*：Web应用发送事件如页浏览量或搜索到Kafka，这样这些事件能够被hadoop实时处理和分析。
* *监控*：实现操作监控的警报和报表，实时的检查日志，发现异常；
* *日志聚合*：Kafka能够用于从多个服务收集日志，然后以一种标准格式提供给多个消费者，包括Hadoop 和Apache Solr；
* *流处理*：类似Spark Streaming能从一个主题读取数据处理然后写入数据到一个新的主题，这样被其他应用再次利用，Kafka的durability（不丢失 持久性）对流处理是非常有用的。