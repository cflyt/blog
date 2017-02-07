---
title: Storm学习
tags: 技术
categories: 未分类
date: 2017-02-07 10:30:27
type:
---

# Storm学习

## 背景

在处理海量数据时，Hadoop是标配的技术框架，Hadoop以批处理的方式处理**离线的数据**，将数据切片计算，使用磁盘作为中间交换的介质。Hadoop不是一个实时计算的框架，不能计算源源不断实时输入的海量数据，并实时给出计算结果。所以在Hadoop之后出现S4，Storm，Spark，Puma等这些实时计算系统，弥补了这个缺陷。这些实时系统框架各有特点，本文介绍近来最流行的Storm框架。

Storm是一个免费开源、分布式、高容错的实时计算系统。Storm令持续不断的流计算变得容易，弥补了Hadoop批处理所不能满足的实时要求。Storm常见的使用场景：

* 流数据处理：Storm可以用来处理源源不断流进来的消息，对这些数据进行实时分析、持续计算，处理之后将结果写入到某个存储中去（Storm不负责存储）。
* 分布式rpc：由于Storm的处理组件是分布式的，而且处理延迟极低，所以可以作为一个通用的分布式rpc框架实现实时业务查询。

比如你需要从日志数据中统计一个论坛过去5分钟讨论最热的帖子，Hadoop计算只能从已经保存在数据库里面日志里面统计，跑个job任务计算结果；如果是Storm，只需把实时的日志流输入到Storm框架中，就能获取统计结果。
你也许有疑问，`消息队列+worker`形式不就可以达到同样的要求么？事实上有很多时候实时统计都是采用这种简便的方式来实现的。但你会发现，如果实时统计需求比较多，处理比较复杂的场景下，系统需要维护一堆消息队列和消费者worker，这构成了非常复杂的系统结构。消费者worker从队列里取消息，处理完成后，去更新数据库，或者给其他队列发新消息。
这样进行实时处理是非常痛苦的。我们主要的精力都花在关注往哪里发消息，从哪里接收消息，消息如何序列化，真正的业务逻辑只占了一小部分。一个应用程序的逻辑运行在很多worker上，但这些worker需要各自单独部署，还需要部署消息队列。最大问题是系统很脆弱，而且不是容错的：需要自己保证消息队列和worker进程工作正常。
Storm完整地解决了这些问题。它是为分布式场景而生的，抽象了消息传递，会自动地在集群机器上并发地处理流式计算，让你专注于实时处理的业务逻辑。

## Storm集群框架
Storm和Hadoop的MapReduce一样，有很好的水平扩展能力，伸缩性很强。它可以像Hadoop那样被部署在多台机器上，实现集群架构。下面是Storm集群的部件图：

![](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2015/09/Snip20150913_2.png)

Storm和Hadoop有很多的相像之处，下面是他们之间的角色对应：

 对应关系 | Hadoop |    Storm
------------ | ------------- | ------------
系统角色|    JobTracker |   Nimbus
系统角色 | TaskTracker |    Supervisor
系统角色| Child | Worker
应用名称    |Job    | Topology
组件接口    | Mapper/Reducer    | Spout/Bolt

在Storm的集群里面有两种节点：控制节点和工作节点；Zookeeper起协调作用，下面是集群中角色的描述：

* Nimbus： 控制节点上面运行一个叫Nimbus进程，Nimbus负责在集群里面分发代码，分配计算任务，并且监控状态。 全局只有一个。
* Supervisor： 每一个工作节点上面运行一个叫做Supervisor进程。Supervisor负责监听从Nimbus分配给它执行的任务，据此启动或停止执行任务的工作进程Worker;
* Zookeeper： Nimbus和Supervisor之间的所有协调工作都是通过Zookeeper集群完成。
* Topology：在Hadoop上面你运行的是MapReduce的Job, 而在Storm上面你运行的是Topology。
* Spout/Bolt: Topology的主要构成，是数据输入，处理的部件。

Nimbus 后台程序和 Supervisor 后台程序都是快速失败(fail-fast)和无状态的，所有状态保存在Zookeeper 或本地磁盘中。
在这种设计中,控制节点并没有直接和工作节点通信,而是借助中介 Zookeeper, 这样一来可以分离**控制节点**和**工作节点**的依赖，将状态信息存放在Zookeeper集群内以快速恢复任何失败的一方。
这意味着你可以 kill 杀掉 Nimbus 进程和 Supervisor 进程，然后重启，它们将恢复状态并继续工作，这种设计使得Storm极其稳定。

## Topology
本章主要介绍Storm中任务的设计思想，各组成部件。
![](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2015/09/Snip20150913_9.png)

### Tuple

Tuple是Storm提供的一个轻量级的数据格式，可以用来包装你需要实际处理的数据。Tuple是一次消息传递的基本单元。一个Tuple是一个命名的值列表，其中的每个值都可以是任意类型的。Tuple本来应该是一个Key-Value的Map，由于各个组件间传递的tuple的字段名称已经事先定义好了，所以Tuple只需要按序填入各个Value，所以就是一个Value List。
一个没有边界的、源源不断的、连续的Tuple序列就组成了Stream。

### Spout
Storm为每个Stream都有一个源头，Spout会从外部读取流数据并发出Tuple。比如Spout可以从消息队列或者数据库中读取消息。Spout可以一次给多个Stream发出数据。

### Bolt
Storm 将流的中间状态转换抽象为Bolt，Bolt可以处理Tuple，同时它也可以发送新的流给其他Bolt使用。一个Bolt可以处理任意数量的输入流，产生任意数量新的输出流。Bolt可以做函数处理，过滤，流的合并，聚合，存储到数据库等操作。Bolt就是流水线上的一个处理单元，把数据的计算处理过程合理的拆分到多个Bolt、合理设置Bolt的task数量，能够提高Bolt的处理能力，提升流水线的并发度。

### Stream Grouping
消息分发策略，即定义一个Stream应该如何分配给Bolt。目前有下面几种分发策略：

* 洗牌分组(Shuffle grouping): 随机分配元组到Bolt的某个任务上，这样保证同一个Bolt的每个任务都能够得到相同数量的元组。
* 字段分组(Fields grouping): 按照指定的分组字段来进行流的分组。例如，流是用字段“user-id"来分组的，那有着相同“user-id"的元组就会分到同一个任务里，但是有不同“user-id"的元组就会分到不同的任务里。
* Partial Key grouping: 跟字段分组一样，流也是用指定的分组字段进行分组的，但是在多个下游Bolt之间是有负载均衡的，这样当输入数据有倾斜时可以更好的利用资源。
* All grouping: 流会复制给Bolt的所有任务。小心使用这种分组方式。
* Global grouping: 整个流会分配给Bolt的一个任务。具体一点，会分配给有最小ID的任务。
* 不分组(None grouping): 说明不关心流是如何分组的。目前，None grouping等价于洗牌分组。
* Direct grouping：直接分组,由Tuple的生产者来定义接收者。
* Local or shuffle grouping：如果目标Bolt在同一个worker进程里有一个或多个任务，元组就会通过洗牌的方式分配到这些同一个进程内的任务里。否则，就跟普通的洗牌分组一样。

通过这些消息分发策略，Storm 解决了组件Spout和Bolt， Bolt和Bolt之间如何发送Tuple的问题。

### Topology
Topology是Storm中最高层次的抽象概念，一个拓扑就是一个流转换图。一个拓扑是一个通过流分组(stream grouping)把Spout和Bolt连接到一起的拓扑结构。图的每条边代表一个Bolt订阅了其他Spout或者Bolt的输出流。一个拓扑就是一个复杂的多阶段的流计算。
一个Storm拓扑跟一个MapReduce的任务(job)概念是类似的。主要区别是：一个MapReduce Job最终会结束， 而一个Topology运永远运行（除非你显式的kill）。

## Storm并发模型
上面介绍了控制节点Nimbus、工作节点Supervisor，工作进程Worker。一个Topology由Nimbus分发到Supervisor，Supervisor**根据这个Topology里面的配置开启数量的Worker进程**，这些Worker进程专门跑这个Topology。Topology里面有部件Spout，Bolt，这些部件可以配置
Executor和Task数量。Executor对应线程，Task对应部件的实例数量。 **Executor的数量平均到每个Worker上，Task的数量平均分到Executor上**。 下面图说明Worker、Executor、Task之间的关系。
![](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2015/09/Snip20150914_15.png)

下面用一个例子说明。

    Config conf = new Config();
    conf.setNumWorkers(2); // use two worker processes

    topologyBuilder.setSpout("blue-spout", new BlueSpout(), 2); // set parallelism hint to 2

    topologyBuilder.setBolt("green-bolt", new GreenBolt(), 2)  // set parallelism hint to 2
                   .setNumTasks(4)
                   .shuffleGrouping("blue-spout");

    topologyBuilder.setBolt("yellow-bolt", new YellowBolt(), 6)
                   .shuffleGrouping("green-bolt");

    StormSubmitter.submitTopology(
            "mytopology",
            conf,
            topologyBuilder.createTopology()
        );

上面代码配置后，得到的拓扑图如下：

![](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2015/09/Snip20150913_13.png)

上面例子中，整个拓扑配置2个Worker进程，那么各个部件并发情况如下：

* Blue Spout：配置parallelism为2，Task数量默认跟parallelism一致，所以2个Task分配到2个Executor，2个Executor分配到2各Worker上。那就是一个Worker运行一个Executor；
* Green Bolt： 设置parallelism为2，Task为4，那么一个Worker运行一个Executor，一个Executor运行2个Task(Bolt的实例)；
* Yellow Bolt：设置parallelism为6，那么一个Worker运行3个Executor，每个Executor运行一个Task。

每个Worker会运行5个Task。

Storm一个灵巧的功能是可以增减worker进程或者executor的数量而不需要重启集群或者拓扑。这种做法叫做rebalancing。有两种方法可以用来做拓扑的rebalance:

* 使用Storm web UI来做
* 使用命令行工具storm rebalance



### Storm单机模式搭建
Storm可以运行在单机上，作为集群模式的一个特例。Storm集群搭建主要包括以下步骤：

1、搭建一个Zookeeper集群

2、在nimbus、supervisor节点安装依赖包

3、在nimbus、supervisor节点下载并解压缩Storm包

4、修改nimbus、supervisor节点的配置文件（storm.yaml）

5、使用storm脚本启动守护进程（包括nimbus、supervisor、ui）

##### 搭建Zookeeper集群
从[Zookeeper官网](https://zookeeper.apache.org/releases.html#download)下载最新版Zookeeper。本文用的是3.4.6版本，zookeeper单机模式非常简单。

    tar zxvf zookeeper-3.4.6.tar.gz
    cd zookeeper-3.4.6
    mv conf/zoo_sample.cfg conf/zoo.cfg
    bin/kServer.sh start

这样就启动了Zookeeper服务。

#### 安装依赖包
本文安装的Storm版本是0.10.0，从0.9版本开始，Storm就支持用netty作为传输信息，默认是zeromq，需要在配置中说明。所以本文不用安装ZeroMQ、JZMQ依赖包。只需要安装java，并且要保证版本大于1.7。否则在启动Storm相关服务，会报错：`Unsupported major.minor version 51.0`，这表示你java的版本过低。
在linux中，可以用apt快捷安装。

    sudo apt-get install openjdk-7-jre

#### 下载并解压缩Storm包
从官网下载稳定发行版，本文为0.10.0.

#### 修改storm.yaml配置
本文修改的配置如下：

    #配置zookeeper
    storm.zookeeper.servers:
        - 127.0.0.1
    storm.zookeeper.port: 2181

    nimbus.host: "127.0.0.1"
    storm.local.dir: "/tmp/storm"
    supervisor.slots.ports:
       - 6700
       - 6701
       - 6702
       - 6703

    #配置ui访问端口
    ui.port: 8080

    #配置netty相关
    storm.messaging.transport: "backtype.storm.messaging.netty.Context"
    storm.messaging.netty.server_worker_threads: 1
    storm.messaging.netty.client_worker_threads: 1
    storm.messaging.netty.buffer_size: 5242880
    storm.messaging.netty.max_retries: 100
    storm.messaging.netty.max_wait_ms: 1000
    storm.messaging.netty.min_wait_ms: 100

#### 启动storm
进入storm的目录，启动下面的进程。

    bin/storm nimbus
    bin/storm supervisor
    bin/storm ui

通过ui查看storm运行状态，在浏览器中输入http://localhost:8080，可以在界面上观察。

### WordCount实例
现在跑一下官方的例子，学习怎么提交Topology。
####安装maven
我们需要用maven进行编译打包，安装maven。

    sudo apt-get install maven

#### 打包Topology例子

    git clone git://github.com/apache/storm.git && cd storm/examples/storm-starter
    mvn package

 这里面我用的是master分支，里面的storm-core版本，在maven的仓库上还没有最新的库，有可能报错：

     [ERROR] Failed to execute goal on project storm-starter: Could not resolve depen
    dencies for project org.apache.storm:storm-starter:jar:0.10.0-SNAPSHOT: Could no
    t find artifact org.apache.storm:storm-core:jar:0.10.0-SNAPSHOT in clojars (http
    s://clojars.org/repo/) -> [Help 1]
checkout 到0.9.3版本的标签即可解决问题。

#### 提交Topology到Storm
在控制节点nimbus上执行

bin/storm jartorm-starter-0.9.3-jar-with-dependencies.jar storm.starter.WordCountTopology test

可以看到console的输出，也可以到ui上看，在Topology栏里面点击test进去查看提交的Topology信息。

![](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2015/09/Snip20150913_14.png)

这个例子是Spout不断的发射英文句子到Bolt，Bolt分解句子，将单词发射到下一个Bolt中处理，最后一个Bolt统计各单词的次数，保存在map里面。


### 参考文献
<https://storm.apache.org/documentation/Tutorial.html>
<http://cloud.berkeley.edu/data/storm-berkeley.pdf>
<http://www.searchtb.com/2012/09/introduction-to-storm.html>
<http://blog.jobbole.com/48595/>
<https://xumingming.sinaapp.com/category/storm/>
<http://www.michael-noll.com/blog/2012/10/16/understanding-the-parallelism-of-a-storm-topology/>
<http://blog.csdn.net/tntzbzc/article/details/19974515>