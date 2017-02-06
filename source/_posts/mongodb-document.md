---
title: MongoDB 文档结构
tags: mongodb
categories: 数据库
date: 2017-02-06 22:37:00
type:
---


## mongoDB: 文档结构

项目上使用了Mongodb作为存储数据库，看了mongodb的一些相关书籍和一些网上资料，现进行总结，以防遗忘。

本文主要介绍文档结构，但本文不会很详细讲mongodb文档的内容，书上已经介绍的很清楚了，而主要介绍的是文档的主要特点，_id的设计原理，mongo使用js，教大家如何查看数据库的document的结构。

### 关系型数据库vs文档型数据库
MongoDB是一种可扩展的高性能的开源的面向文档（document-oriented ）的数据库。

*   关系数据库比如 MySQL，通常将不同的数据划分为一个个“表”，表的数据是按照“行”来储存的。而关系数据库的“关系”是指通过“外键”将表间或者表内的数据关联起来。每行的属性（列）是固定的，每行都拥有一样的属性。

*   在 MongoDB 中，文档是以 BSON 格式（类似 JSON）储存的，可以支持丰富的层次的结构。MongoDB 中的文档其实是没有模式的，不像 SQL 数据库那样在使用前强制定义一个表的每个字段。这样可以避免对空字段的无谓开销，可以在开发中根据需求增加或者删除字段，灵活设计。在 MongoDB 中，一个数据项叫做Document，一个文档可以嵌入另一个文档，代表一种弱的关联,多个文档组成的集合，就叫Collection。

下面是关系型数据库和文档数据库的结构图。

![](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2015/01/relation-scheme2.png "关系型数据库")

![](http://iamsummer-wordpress.stor.sinaapp.com/uploads/2015/01/norelation-scheme2.png "关系型数据库")

上面这两张图很好的说明了关系型数据库和文档型数据库表结构的区别，上面的两张图代表博客、作者、评论、标签的实体以及它们之间的关系。

在图1中数据关系范式化后，User、Blog、Blog Entry、Tag、Comment各存储为一张表，Blog Entry中分别引用了User、Blog的键作为外键，表示它们之间的关系，而comment引用了User、Blog Entry的键作为外键；Tag引用Blog Entry的键；
在图2中有两个collection，User Collection和Blog Collections。Blog Collection中存储了Author字段，根据这个字段可以在User Collection中找到相关的记录，同时，在Comment也存储了Comment By字段，指向User Collection的字段。而Blog Entry、Comment作为内嵌的文档存储同一个collection中。

关系型数据库和mongodb文档型数据库，有一些概念类似。


| RDBMS | document DB |
| ------| ----------- |
| Table(表）) | collection（集合）  |
| Index(索引) | Index (索引)  |
| Join(连接)  | Embedded（内嵌）|
| Partition(分区)   | Shard(分片)  |
| Partition key |  Shard Key |

### _id
在一个特定集合内部，需要唯一的标识文档。因此MongoDB中存储的文档都由一个”_id”键，用于完成此功能。这个键的值可以是任意类型的，默认是ObjectId对象。ObjectId对象的生成思路是本文的主题之一，也是很多分布式系统可以借鉴的思路。

为了考虑分布式，“_id”要求不同的机器都能用全局唯一的同种方法方便的生成它。因此不能使用自增主键（需要多台服务器进行同步，既费时又费力），下面介绍设计原理。

ObjectId使用12字节的存储空间，其生成方式如下：

0 1 2 3 | 4 5 6 | 7 8 | 9 10 11
------  | ------ | ---- | -------
时间戳   | 机器ID     | PID   | 计数器


1. 前四个字节时间戳是从标准纪元开始的时间戳，单位为秒，有如下特性：

    * 时间戳与后边5个字节一块，保证秒级别的唯一性；
    * 保证插入顺序大致按时间排序；
    * 隐含了文档创建时间；

2. 机器ID是服务器主机标识，通常是机器主机名的散列值。

3. 同一台机器上可以运行多个mongod实例，因此也需要加入进程标识符PID。

4. 前9个字节保证了同一秒钟不同机器不同进程产生的ObjectId的唯一性。
后三个字节是一个自动增加的计数器（一个mongod进程需要一个全局的计数器），保证同一秒的ObjectId是唯一的。同一秒钟最多允许每个进程拥有（256^3 = 16777216）个不同的ObjectId。

总结一下：时间戳保证秒级唯一，机器ID保证设计时考虑分布式，避免时钟同步，PID保证同一台服务器运行多个mongod实例时的唯一性，最后的计数器保证同一秒内的唯一性（选用几个字节既要考虑存储的经济性，也要考虑并发性能的上限）。

“_id”既可以在服务器端生成也可以在客户端生成，在客户端生成可以降低服务器端的压力。很多驱动都是在客户端生成后传输到服务端的，mongodb的思想是能在客户端生成的，就不要在服务端做。

**ps**：_id已经包含时间戳信息，记录生成时间可以由此获得。

## mongo运行js

mongo是一个简化的javascript shell，启动mongo时就可以使用命令操作数据库。mongodb所支持的运行js代码的场景：

*   --eval参数
    在启动mongo时使用--eval参数，就可以在运行指定js代码

    >mongo -u name -p password 127.0.0.1:30000/gacc --eval "printjson(db.getCollectionNames())"

    上面的js代码打印gacc数据库的所有collection

*   js文件
    *   连接mongo时，可以将js文件放到后面执行。

        mongo -u name -p password 127.0.0.1:30000/gacc jsfile1.js jsfile2.js

    *  或者连接mongo后，用load的命令执行

        load("scripts/test.js")
        load("/home/gzchenfei/Temp/scriptis/test.js")
        load支持相对路径和绝对路径。
*   mapReduce
    mongodb支持mapReduce框架，而所使用的语法为js的语法。
*   $where
    使用$where查询时，传递js函数和js代码的字符。查询能力很强，但是效率慢，斟酌使用。

## 利用脚本查看mongodb的文档结构
为什么要介绍使用js，目的是为了介绍一个脚本variety.js。
当你接手一个用mongodb存数据的项目，没有相关的说明文档，你要了解集合存储的数据结构，你可能会花大量时间跟前同事沟通，而且可能有些文档结构字段了解不全。
variety.js这是一个文档分析脚本，可以分析文档的字段组成以及字段的类型；可以分析一个字段在全文档中出现的比例；可以发现不常用的字段；可以发现嵌套的结构对象，以及嵌套的深度。

### 基本用法
命令
>mongo -u uutest -p xxxx 127.0.0.1:30000/gacc --eval "var collection = 'game_info'" variety.js

`var collection = game_info`指定要检索的数据集为game_info，得到结果如下：


    +---------------------------------------------------------------------------+
    | key                    | types         | occurrences | percents           |
    | ---------------------- | ------------- | ----------- | ------------------ |
    | _id                    | ObjectId      | 162         | 100                |
    | icon_url               | String        | 162         | 100                |
    | name                   | String        | 162         | 100                |
    | proc_name              | String        | 162         | 100                |
    | search_key             | String        | 162         | 100                |
    | search_string          | String        | 162         | 100                |
    | type                   | String        | 162         | 100                |
    | updateTime             | Object,Number | 162         | 100                |
    | updateTime.floatApprox | Number        | 161         | 99.38271604938271  |
    | desc                   | String        | 144         | 88.88888888888889  |
    | netbar                 | Number        | 111         | 68.51851851851852  |
    | priority               | Number        | 13          | 8.024691358024691  |
    | forbidden_vpn_type     | String        | 10          | 6.172839506172839  |
    | support_games          | String        | 4           | 2.4691358024691357 |
    | delay_timer            | Number        | 2           | 1.2345679012345678 |
    | default_game_server_id | ObjectId      | 1           | 0.6172839506172839 |
    +---------------------------------------------------------------------------+

上面key这一列代表文档中出现过的字段，type表示字段的类型，occurrences出现的文档次数，percents表示占比。这些数据一目了然，知道哪些字段经常使用，哪些字段不常用，从而对collection的数据结构有个整体的感知。

variety.js是使用了map reduce的方式分析的，mongodb的map reduce计算框架效率很低，在数据量很大的情况下，等待结果的时间很长，所以最好是设置从副本集中读取。

### limit限制
如果你对太旧的记录不感兴趣，你还可以使用`limit`来限制，使用limit表明只分析最新的记录的字段，在整个集合中的分布。
>mongo -u uutest -p xxxx 127.0.0.1:30000/gacc --eval "var collection = 'game_info', limit=1" variety.js

    +----------------------------------------------------------------------+
    | key                    | types    | occurrences | percents           |
    | ---------------------- | -------- | ----------- | ------------------ |
    | _id                    | ObjectId | 162         | 100                |
    | icon_url               | String   | 162         | 100                |
    | name                   | String   | 162         | 100                |
    | proc_name              | String   | 162         | 100                |
    | search_key             | String   | 162         | 100                |
    | search_string          | String   | 162         | 100                |
    | type                   | String   | 162         | 100                |
    | updateTime             | Object   | 162         | 100                |
    | netbar                 | Number   | 111         | 68.51851851851852  |
    +----------------------------------------------------------------------+

上面使用了limit=1进行限制，只分析最新一个记录所有的可以在整个collection中的出现占比。

###分析嵌入文档
variety.js还可以分析嵌套的文档以及深度。
在mongodb中插入测试文档
>db.test.insert({name:"chris", embedObject:{a:{b:{c:{d:{e:1}}}}}});

使用variety.js分析结构

>mongo -u uutest -p xxx 127.0.0.1:30000/gacc_stat --eval "var collection = 'test'" variety.js

结果如下:

    +-----------------------------------------------------------+
    | key                   | types    | occurrences | percents |
    | --------------------- | -------- | ----------- | -------- |
    | _id                   | ObjectId | 1           | 100      |
    | embedObject           | Object   | 1           | 100      |
    | embedObject.a         | Object   | 1           | 100      |
    | embedObject.a.b       | Object   | 1           | 100      |
    | embedObject.a.b.c     | Object   | 1           | 100      |
    | embedObject.a.b.c.d   | Object   | 1           | 100      |
    | embedObject.a.b.c.d.e | Number   | 1           | 100      |
    | name                  | String   | 1           | 100      |
    +-----------------------------------------------------------+

分析的深度默认为99层，可以使用maxDepth进行限制。
>mongo -u uutest -p xxx 127.0.0.1:30000/gacc_stat --eval "var collection = 'test'，maxDepth=3" variety.js

上面限制解析深度为3，结果如下：

    +-----------------------------------------------------+
    | key             | types    | occurrences | percents |
    | --------------- | -------- | ----------- | -------- |
    | _id             | ObjectId | 1           | 100      |
    | embedObject     | Object   | 1           | 100      |
    | embedObject.a   | Object   | 1           | 100      |
    | embedObject.a.b | Object   | 1           | 100      |
    | name            | String   | 1           | 100      |
    +-----------------------------------------------------+

### 其他限制条件

如果你只对对某部分文档感兴趣，可以使用`query`限制查询的范围。

>mongo -u uutest -p xxx 127.0.0.1:30000/gacc_stat --eval "var collection = 'game_info', query={'netbar': 0}" variety.js


`query`查询条件，语法跟mongo的js语法一样。

同样，你还可加入`sort={'timestamp': -1}`限制,分析时按照指定的顺序分析每个文档。
除了上面的输出格式，限定`outputFormat='json'`.结果显示为json格式。

variety.js源码可以从 [github](https://github.com/variety/variety "github")上下载。

## 文档一些限定

 1.BSON文件的最大限制为 单个文档不超过16MB。

最大文件大小的限制有助于确保单个文件不会过多的占用内存，在传输过程中,不会占用过多的宽带。当需要存储到MongoDB中的单个文件大小超过了Mongo DB限制的最大文件大小时,MongoDB 提供了另外一套API—Grid FS 。

 2.深度嵌套BSON 文档

自Mongo DB 2.2版本更新之后, MongoDB 支持单个文件不超过100的BSON文件嵌套。

3.命名空间的长度：在每一个命名空间中,包括数据库名，集合名称在内,不得超过123KB。
4.mongodb一次接受的消息不得大于48m，如果大于48m,驱动会把内容分成多个48m进行传输。