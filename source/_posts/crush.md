---
title: crush算法
tags: 技术
categories: 未分类
date: 2017-02-07 10:37:48
type:
---

# crush算法

分布式存储元数据在实现过程中主要有两类设计：中心化和去中心化。
中心化：元数据存储在一个中心的服务器中，客户端增加数据时，中心服务器进行分布数据和记录数据的信息，查询数据时，从中心服务器获取位置信息，读取内容；
去中心化：中心服务器存储少量的元数据，某个数据的分布由客户端通过一个分布算法计算得到；

中心化的设计的优点就是管理简单，缺点是中心节点容易成为瓶颈；去中心化刚好相反，优点是将压力分散到各存储节点，无单点故障，缺点是管理复杂，存储结构变化时数据迁移困难；

ceph是一个开源的分布式存储系统，具有高性能，高可靠性、可扩展性，使用crush数据分布算法去中心化，在客户端计算数据位置，可以具体到某个磁盘，某个机器，某个机架，甚至是某个数据中心，这样就可以考虑到机房、机架、机器这样的存储层次，在每层设置不同的存储策略，从而达到较好的分布效果。而且使用该算法在存储设备增加删除时，只有小部分的数据需要移动，减少迁移成本。

本文将介绍ceph的核心数据分布算法-crush。

## 一致性哈希

一致性哈希算法是分布式系统中常用的算法。比如，一个分布式的存储系统，要将数据存储到具体的节点上，如果采用普通的hash方法，将数据映射到具体的节点上，如key%N，key是数据的key，N是机器节点数，如果有一个机器加入或退出这个集群，则所有的数据映射都无效了，如果是持久化存储所有的数据都要迁移，如果是分布式缓存，则其他缓存就失效了。一致性哈希算法将存储对象映射到一个环上，当存储服务器数量发生变化时，只影响相邻的服务器，规避了大规模的数据震荡和存储映射失效。crush算法有点类似一致性哈希，但是一致性哈希没有考虑的存储设备的层次性，不能对存储设备进行加权控制。

crush算法目的是使数据能够根据设备的存储能力加权控制并保持一个相对的概率平衡地分布，并且在设备增加或退出集群时，较优的使迁移的数据量少。副本放置在具有层次结构的存储设备中，这对数据安全也有重要影响。crush算法通过集群的拓扑信息，副本放置策略可以将数据对象独立在不同故障域，同时仍然保持所需的分布。例如，为了防止可能的并发故障，应该确保设备上的数据副本放置在不同的机架、主机、电源、控制器、或其他的物理位置。

## crush算法描述

crush需要输入key值X、cluster map(描述存储集群的层级结构)、和数据分布规则(rule)。分布算法由描述存储集群的层级结构cluster map控制。这个map可以这样描述：集群有多个数据中心，每个数据中心有不同机房构成，机房又有机架，机架装满服务器，服务器装满磁盘。数据分配的策略是由定位规则来定义的，定位规则指定了集群中将保存多少个副本，以及数据副本的放置有什么限制。例如，可以指定数据有三个副本，这三个副本必须放置在不同的机柜中，使得三个数据副本不公用一个物理电路。每一层的设备还可以分配一个权重，crush算法根据种每个设备的权重尽可能概率平均地分配数据。

下图为存储拓扑结构：
![](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2016/02/Snip20160216_2.png)


给定一个输入x，crush算法将输出一个确定的有序的储存目标向量R。当输入x，crush利用多重整数hash函数根据集群map、分配规则（rule）、以及x计算出独立的完全确定可靠的映射关系。crush分配算法是伪随机算法，并且输入的内容和输出的储存位置之间是没有显式相关的。我们可以说crush算法在集群设备中生成了“伪集群”的数据副本。调用过程描述如下：

```
• CRUSH(X)  ->  R(osdn1, osdn2, osdn2)
• 参数
    – X input key
    – Hierachical Cluster map：可用存储资源层级结构（有多少机架，每个机架上有多少服务器）
    – Placement rules：每个对象有多少个副本，副本分配限制条件，比如3个副本在不同的机架上
• 输出一组存储目标集合
```

## Cluster map
Ceph将系统的所有硬件资源描述成一个树状结构，由device和bucket组成，它们都有id和权重值。bucket是每层存储的抽象，可以是（机房，机架），device是叶子节点。Bucket可以包含任意数量item，表示子节点。item可以都是的devices或者都是buckets。管理员控制存储设备的权重。权重和存储设备的容量有关。Bucket的权重被定义为它所包含所有item的权重之和。crush基于4种不同的bucket选择类型，每种有不同的选择算法。
bucket的描述如下：

```
[bucket-type] [bucket-name] {
    id [a unique negative numeric ID]
    weight [the relative capacity/capability of the item(s)]
    alg [the bucket type: uniform | list | tree | straw ]
    hash [the hash type: 0 by default]
    item [item-name] weight [weight]
}

```

```
rack rack0 {
        id -5           # do not change unnecessarily
        weight 8.000
        alg uniform     # do not change bucket size (4) unnecessarily
        hash 0  # rjenkins1
        item host0 weight 2.000 pos 0
        item host1 weight 2.000 pos 1
        item host2 weight 2.000 pos 2
        item host3 weight 2.000 pos 3
}
```

id为负数，以便与device区分;
weight：bucket的权重，为子节点权重之和;
alg：bucket的算法类型(uniform, list, tree, straw),选择子节点的方式；
hash：bucket中使用到的hash算法；
item：bucket里包含哪些元素，即子节点；

## rule
有了crush map之后，如何一步一步从bucket中选出元素，这个就是有rule来定义的。一个rule就是一系列的操作。
一个rule的定义是这样的：

```
rule <rulename> {
    ruleset <ruleset>     //rule id
    type [ replicated | raid4 ]
    min_size <min-size>   //备份最小数
    max_size <max-size>     //备份最大数
    step take <bucket-type>     //从bucket-type开始挑选
    step [choose|chooseleaf] [firstn|indep] <N> <bucket-type>   //挑选规则
    step emit
}
```
```
rule  replicated_ruleset {
    ruleset 1
    type replicated
    min_size 1
    max_size 10
    step take default
    step choose firstn 2 type rack
    step chooseleaf firstn 0 type host
    step emit
}
```
ruleset：id，表明这个rule是属于这个ruleset的。
type：表明这个rule在哪使用，storage drive (replicated) or a RAID。
min_size和max_size用来限定这个rule的使用范围，当指定副本数小于min_size或者大于max_size的时候，不使用此条rule。
step take <bucket-name>：从这个bucket开始往下遍历挑选；
step choose firstn {num} type {bucket-type}：选择n个指定类型的bucket，num通常是副本的数量。 此条语句可以有多个；

* num==0，表示选择副本数量的bucket；
* num>0, 表示选择num个bucket；
* num<0, 表示选择r-num个bucket；

step chooseleaf firstn {num} type {bucket-type}： 表示选择指定数量的指定类型的bucket，从这些bucket中往下遍历获取一个叶子节点；数量的指定同上；
step emit： 输出当前选择的值；

## crush算法原理
crush通过配置的map和rule，进行数据对象及其副本的分配放置。算法的伪代码如下图：
![](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2016/02/crush_alg.jpg)

每个rule都包含上述伪代码的操作。crush函数的整型输入参数就是一个典型的存储对象名或者标示符。

* 操作take(a)选择了一个在存储层次的bucket并把这个bucket的子item赋值向量i，这是为后面的操作做准备。
* 操作select(n,t)，迭代操作每个item(向量i中的item)，对于每个item(向量i中的item)向下遍历(遍历这个item所包含的子item)，都返回n个不同的item(type为t的item)，存储设备有一个绑定类型（例如rows，cabinets）。并把这些item都放到向量i中。作为随后被调用的select(n,t)操作的输入参数或者进行输出。select函数会调用c(r, x)函数，这个函数会在每个bucket中伪随机选择一个item。
* emit：把向量i放到result中。

例如下图，有4层存储结构的图，存储3个副本；存储策略为从4个机排中随机选择一排，再从这排中选择出3个机柜，从机柜中选择其中一台主机下面的某个盘进行存储。
![](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2016/02/Snip20160216_3.png)

对应关系如下：

rule | Action | result
---|----|-----
step take root | take(root) | root
step choose firstn 1 type row | select(1, row) | row2
step choose firstn 3 type cabinet | select(3, cabinet) | cab21 cab23 cab24
step choose firstn 1 type disk | select(1, disk) | disk2107 disk2313 disk2437

###冲突故障过载
select(n, t)操作会循环选择第 r=1,…,n 个副本，r作为选择参数。在这个过程中，假如选择到的item遇到三种情况(冲突，故障，超载)时，crush会拒绝选择这个item。

* 冲突：这个item已经在向量i中，已被选择。
* 故障：设备发生故障，不能被选择。
* 超载：设备使用容量超过警戒线，没有剩余空间保存数据对象。

对于故障和超载，ceph会进行标记，并重启select重新选择；对于冲突情况，算法先在Local域（目标type的父节点孩子节点）内重新选择，尝试有限次数后，如果仍然找不到满足条件的Bucket，那就回到Descent域（当前bucket的子树）重新选择。


### bucket算法类型

crush映射算法解决了效率和扩展性这两个矛盾的目标。而且当存储集群发生变化时，可以最小化数据迁移，并重新恢复平衡分布。crush定义了四种具有不同算法的的buckets。每种bucket基于不同的数据结构，并有不同的c(r,x)伪随机选择函数。

不同的bucket有不同的性能和特性：

* Uniform Buckets：适用于具有相同权重的item，而且bucket很少添加删除item。增加删除机器后所有数据需要重新分布，但是它的查找速度是最快的。

* List Buckets：它的结构是链表结构，所包含的item可以具有任意的权重。crush从表头开始查找副本的位置，它先得到表头item的权重Wh、剩余链表中所有item的权重之和Ws，然后根据hash(x, r, i)得到一个[0~1]的值v，假如这个值v在 [0~Wh/Ws)之中，则副本在表头item中，并返回表头item的id，i是item的id号。否者继续遍历剩余的链表。 这类bucket增加机器时移动的数据是最优的，减少机器时需要数据重新分布。

* Tree Buckets：链表的查找复杂度是O(n)，决策树的查找复杂度是O(log n)。item是决策树的叶子节点，决策树中的其他节点知道它左右子树的权重，节点的权重等于左右子树的权重之和。crush从root节点开始查找副本的位置，它先得到节点的左子树的权重Wl，得到节点的权重Wn，然后根据hash(x, r, node_id)得到一个[0~1]的值v，假如这个值v在[0~Wl/Wn)中，则副本在左子树中，否者在右子树中。继续遍历节点，直到到达叶子节点。Tree Bucket的关键是当添加删除叶子节点时，决策树中的其他节点的node_id不变。决策树中节点的node_id的标识是根据对二叉树的中序遍历来决定的(node_id不等于item的id，也不等于节点的权重)。

* Straw Buckets：这种类型让bucket所包含的所有item公平的竞争(不像list和tree一样需要遍历)。这种算法就像抽签一样，所有的item都有机会被抽中(只有最长的签才能被抽中)。每个签的长度是由length = f(Wi)*hash(x, r, i) 决定的，f(Wi)和item的权重有关，i是item的id号。c(r, x) = MAX(f(Wi) * hash(x, r, i))。这种bucket增加删除机器时的数据移动量都是最优的。

各bucket算法比较：

Action | Uniform | List | Tree | Straw
--|--|--|--|--
时间复杂度|O(1)|O(n)|O(lgn)|O(n)
增加节点 | 差 | 优 | 好 | 优
删除节点 | 差 | 差 | 好 | 优


ceph默认是使用straw类型的bucket，可以根据实际情况进行选择。

## crushtool

安装ceph系统后，就自带了一个工具crushtool用来测试crush算法，可以创建存储结构map、测试规则rule，模拟数据输入。

### 创建crushmap
我们组建一个存储结构，假如有有2个机架rack， 每个rack下面有2台host， 每个host有2块磁盘，则一共有8个磁盘。
我们的bucket有三种: root、rack、host。root包含的item是rack，root选择rack的算法是straw。rack包含的item是host，rack选择host的算法是tree。host包括的item是device，host的选择device的算法uniform。这是因为每个host包括的device的数量和权重是一定的，不会改变，因此要为host选择uniform结构，这样计算速度最快。
执行命令：

```
crushtool --num_osds 8 -o crushmap --build host uniform 2 rack tree 2 root straw 0
结果显示：
# id    weight  type name   reweight
-7  8   root root
-5  4       rack rack0
-1  2           host host0
0   1               osd.0   1
1   1               osd.1   1
-2  2           host host1
2   1               osd.2   1
3   1               osd.3   1
-6  4       rack rack1
-3  2           host host2
4   1               osd.4   1
5   1               osd.5   1
-4  2           host host3
6   1               osd.6   1
7   1               osd.7   1
```

### 编辑 crush map
创建之后会在本地目录生成一个crush map的二进制文件，我们可以通过crushtool工具进行反编译。

```
crushtool -d crushmap -o map.txt
```
打开map.txt得到：

```
# begin crush map
tunable choose_local_tries 0
tunable choose_local_fallback_tries 0
tunable choose_total_tries 50
tunable chooseleaf_descend_once 1

# devices
device 0 device0
device 1 device1
device 2 device2
device 3 device3
device 4 device4
device 5 device5
device 6 device6
device 7 device7

# types
type 0 device
type 1 host
type 2 rack
type 3 root

# buckets
host host0 {
        id -1           # do not change unnecessarily
        # weight 2.000
        alg uniform     # do not change bucket size (2) unnecessarily
        hash 0  # rjenkins1
        item device0 weight 1.000 pos 0
        item device1 weight 1.000 pos 1
}
host host1 {
        id -2           # do not change unnecessarily
        # weight 2.000
        alg uniform     # do not change bucket size (2) unnecessarily
        hash 0  # rjenkins1
        item device2 weight 1.000 pos 0
        item device3 weight 1.000 pos 1
}
host host2 {
        id -3           # do not change unnecessarily
        # weight 2.000
        alg uniform     # do not change bucket size (2) unnecessarily
        hash 0  # rjenkins1
        item device4 weight 1.000 pos 0
        item device5 weight 1.000 pos 1
}
host host3 {
        id -4           # do not change unnecessarily
        # weight 2.000
        alg uniform     # do not change bucket size (2) unnecessarily
        hash 0  # rjenkins1
        item device6 weight 1.000 pos 0
        item device7 weight 1.000 pos 1
}

rack rack0 {
        id -5           # do not change unnecessarily
        # weight 4.000
        alg tree        # do not change pos for existing items unnecessarily
        hash 0  # rjenkins1
        item host0 weight 2.000 pos 0
        item host1 weight 2.000 pos 1
}
rack rack1 {
        id -6           # do not change unnecessarily
        # weight 4.000
        alg tree        # do not change pos for existing items unnecessarily
        hash 0  # rjenkins1
        item host2 weight 2.000 pos 0
        item host3 weight 2.000 pos 1
}
root root {
        id -7           # do not change unnecessarily
        # weight 8.000
        alg straw
        hash 0  # rjenkins1
        item rack0 weight 4.000
        item rack1 weight 4.000
}

# rules
rule replicated_ruleset {
        ruleset 0
        type replicated
        min_size 1
        max_size 10
        step take root
        step chooseleaf firstn 0 type host
        step emit
}

# end crush map
```
通过这个文件可以修改设备的权重值，alg的类型等；

#### 编辑数据分配rule
在map.txt中可以看到默认生成一条replicated_ruleset规则。

```
step take root
step chooseleaf firstn 0 type host
step emit
```

这3条step表明从root开始，在不同的主机上分配指定数量的副本。
我们将机架考虑进去，将数据分配到不同的机架上面，将规则进行改写：

```
step take root
step choose firstn 0 type rack
step chooseleaf firstn 1 type host
step emit
```

#### 编译map
将编辑后的map.txt进行编译程二进制文件。

```
 crushtool  -c map.txt  -o map.bin
 ```

#### 模拟测试
```
crushtool -i map.bin  --test --show-statistics --rule 0 --min-x 1 --max-x 5 --num-rep 2

rule 0 (replicated_ruleset), x = 1..5, numrep = 2..2
CRUSH rule 0 x 1 [4,0]
CRUSH rule 0 x 2 [5,0]
CRUSH rule 0 x 3 [7,3]
CRUSH rule 0 x 4 [6,1]
CRUSH rule 0 x 5 [4,0]
rule 0 (replicated_ruleset) num_rep 2 result size == 2: 5/5
```
* --test: 表明调用crushtool的模拟输入功能测试；
* --rule： 使用那条规则；
* --min-x： 输入的X的起始值；
* --max-x：输入的X的终点值；
* --num-rep： 需要的副本数量；
* -i： 二进制的map文件
* --show-statistics： 显示结果

上面的测试中输入X为1到5， 指定备份数量为2， 对应不同的X值得到分配结果。
从结果可以看出，两个副本落在不同的机架的不同主机上面。

## 总结
本文简单的解释了ceph分布式文件存储系统中的核心-数据分配选择算法crush，此算法对传统的算法进行改进，而且适用性比较广，实用性强，可以用在生产环境中。


## 相关资料：
https://access.redhat.com/webassets/avalon/d/Red_Hat_Ceph_Storage-1.2.3-Storage_Strategies-en-US/Red_Hat_Ceph_Storage-1.2.3-Storage_Strategies-en-US.pdf
http://www.crss.ucsc.edu/media/papers/weil-sc06.pdf
http://www.wzxue.com/ceph-storage/
https://github.com/ceph/ceph/tree/master/src/crush
