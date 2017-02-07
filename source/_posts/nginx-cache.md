---
title: nginx缓存
tags: 技术
categories: 未分类
date: 2017-02-07 10:44:40
type:
---

## nginx缓存

最近调研了目前web的缓存方案，用以减缓瞬间飙升并发访问给后端服务带来的压力。在几个流行的开源的web缓存技术中，发现nginx本来就很好的支持缓存功能，而且在最新的1.9.8版上加强了对Byte-Range特性的支持，nginx是不错的选择。本文就nginx支持的缓存进行使用说明。

### 缓存方式
nginx支持两种缓存方式，一种是文件缓存；一种是内存缓存；
其中文件缓存方式依赖于内置的模块proxy cache（uwsgi对应有uwsgi cache）； 另一种是srcache+memcache/redis方案。本文先介绍proxy cache。

#### 1 proxy cache
proxy cache开启很简单，直接上简单配置文件：

    proxy_cache_path /path/to/cache levels=1:2 keys_zone=my_cache:100m max_size=10g inactive=60m;

    server {
    ...
        location / {
            proxy_cache my_cache;
            proxy_pass http://my_upstream;
        }
    }

参数说明：

* `proxy_cache_path`: 文件保存目录；
* `levels=1:2`: 表示设置两层缓存目录，缓存目录的第一级目录是1个字符，第二级目录是2个字符，即/path/to/cache/a/1b这种形式。一个目录中文件太多，会导致文件的访问性能下降。
*  `keys_zone=my_cache:100m`: 表示缓存池的名字为my_cache, 大小为100M，这个缓存池用来缓存文件的key，缓存池是分配在内存的。如果一个文件的key占的空间是100字节，那100M的内存大概可以存1百万个文件的key。
*  `max_size=10g`: 缓存文件的总大小不能超过10G；
*  `inactive=60m`: 缓存的文件如果超过60分钟没有被访问，那么会被清除；

`proxy_cache my_cache` 指定了使用哪个缓存空间，可以在location下面配置，仅对当前目录有效；也可以在server下面配置，全局有效。

上面就是基本的配置，能满足了基本的要求，我们进一步优化场景配置。

     location / {
            proxy_cache_key $host$uri$args;
            proxy_cache_revalidate on;
            proxy_cache_min_uses 3;
            proxy_cache_lock on;
            proxy_cache_lock_age 200s;
              .....
        }

* `proxy_cache_key`: 指定cache缓存的key组成，nginx把这个字符串md5值作为缓存的key。
* `proxy_cache_revalidate`: 开启这个选项，当用户请求一个Cache-Control头已经过期的文件，nginx会携带 If-Modified-Since 头，请求在这个时间更新之后的文件。这个选项有利于减少传输带宽。
* `proxy_cache_min_uses`: 最少访问多少次才进行缓存，该选项帮助nginx判断该文件是否是热点文件。
* `proxy_cache_lock`: 多个请求访问同一个文件时，只允许第一个请求访问，等待第一个请求缓存后，后面的请求从缓存中拉取数据。
* `proxy_cache_lock_age 200s`: 锁的时间，默认为5s，一般要把这个时间设置大于文件读取时间，如果超过这个时间还没缓存好，那么nginx再次发起多个请求后端服务器。


### 1.1 出错时处理
    location / {
        ...
        proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
    }
使用该配置当程序中遇到error，超时，500、502、503、504时，返回当前老版本缓存，这样做总比报错强。
updating： 当访问一个缓存文件时，恰巧这个文件更新缓存，此时该访问返回旧文件，而不是等待更新完访问新缓存。

### 1.2 多个缓存目录
可以使用多个磁盘增加磁盘的吞吐量。

    proxy_cache_path /path/to/hdd1 levels=1:2 keys_zone=my_cache_hdd1:10m max_size=10g
                     inactive=60m use_temp_path=off;
    proxy_cache_path /path/to/hdd2 levels=1:2 keys_zone=my_cache_hdd2:10m max_size=10g
                     inactive=60m use_temp_path=off;

    split_clients $request_uri $my_cache {
                  50%          “my_cache_hdd1”;
                  50%          “my_cache_hdd2”;
    }

    server {
        ...
        location / {
            proxy_cache $my_cache;
            proxy_pass http://my_upstream;
        }
    }

split_clients模块将请求按配置的比例，分发到两个缓存空间。

### 1.3 忽略Cache-Control
    location /images/ {
        proxy_cache my_cache;
        add_header X-Cache-Status $upstream_cache_status;
        proxy_ignore_headers Cache-Control;
        proxy_cache_valid 200 206 30m;
        ...
    }

如果用户想知道此请求有没有缓存名字，可以添加一个头返回给用户，值为`upstream_cache_status`。
默认情况下，proxy cache只缓存应答中http头 Cache-Control 成立的请求；如果需要缓存所有请求，需要配置忽略Cache-Control头，并配置proxy_cache_valid缓存时间。


### 1.4 Byte-Range流的支持
对于一个大文件，如果用户请求中指定了Byte-Range， 只需要返回文件的某一部分。proxy cache是怎么做的呢？
在nginx1.9 以前，nginx会先请求源服务器，把整个文件缓存下来，然后从缓存中读取需要的部分返回给用户，用户在此过程中处于一直挂起状态。

想象在访问量大的场景下会怎样呢，在读取这个大文件时，会同时有很多的请求发给后端读取这个文件，如果锁相继超时（看下面锁配置的说明）那么一次缓存都没有成功，这将导致越来越多的请求堆积，后果是把服务器给压垮了。

解决方式有两种：

1. 使用proxy_cache_lock，并合理配置各项锁超时的时间，下面是一个sample：

        location /images/ {
             proxy_cache_lock on;

            # Immediately forward requests to the origin if we are filling the cache
            proxy_cache_lock_timeout 0s;

            # Set the 'age' to a value larger than the expected fill time
            proxy_cache_lock_age 200s;

            proxy_cache_use_stale updating;
        }
    * `proxy_cache_lock on`: 当多个请求访问同一个文件时，只有第一个请求进行访问后端服务器，其他请求阻塞，第一个请求完成并进行缓存后，一并处理后面的请求。
    * `proxy_cache_lock_timeout`： 配置为0秒，表示在锁超时后，向源服务器发送原始请求，即带Range头部，结果返回后不进行缓存。
    * `proxy_cache_lock_age`：一般设置比缓存填充时间要大，超过这个时间，nginx再次发起多个请求给源服务器。
2. 在nginx 1.9.8以上版本，使用slice模块来解决此问题。

        server {
            ....

            proxy_cache mycache;

            slice              1m;
            proxy_cache_key    $host$uri$is_args$args$slice_range;
            proxy_set_header   Range $slice_range;
            proxy_http_version 1.1;
            proxy_cache_valid  200 206 1h;

            location / {
                proxy_pass http://origin:80;
            }
        }

    * `slice`: 设置分片的大小；
    * `proxy_cache_key`: 需要把$slice_range字段加入到key内容中，不然key就没法区分是哪一段了；
    * `proxy_http_version`: Byte-Range只有http 1.1版本才支持；
    * `proxy_cache_valid`: 使用Byte-Range返回的状态码为206，确定缓存返回为206的请求。

    举例说明，假如存储的文件为500B，slice为100B，用户发起请求Range: bytes=10-50; nginx根据slice的设置，并且10-50在第一个slice内，那么nginx向源服务器发起Range: bytes=0-100获取slice, 源服务器返回后nginx将10-50的范围给用户，并将slice的内容进行缓存，以备下次用。
    如果用户请求的range跨越了多个slice， nginx发起多个请求请求符合范围的slice，然后将从多个slice中组装内容返回给用户。

### 1.5缓存的清除
文件缓存了，要如何进行清除呢，nginx也提供了purge模块，不过这个模块不是内置的，需要重新编译的，加入参数--add-module=/usr/local/ngx_cache_purge，我们可以手动进行删除文件，下面是针对两层目录`levels=1:2`的删除方式：

    先根据proxy_cache_key指定的字段内容计算文件的key
    $ echo -n "http://hostname/path/to/test.html" | md5sum
    48a17b3843b4a5a6314c95d610c5be6d  -
    获取文件路径
    $ echo 48a17b3843b4a5a6314c95d610c5be6d | awk '{print "/path/to/cache/"substr($1,length($1),1)"/"substr($1,length($1)-2,2)"/"$1}'
    /path/to/cache/d/e6/48a17b3843b4a5a6314c95d610c5be6d
    删除文件
    $ rm /path/to/cache/d/e6/48a17b3843b4a5a6314c95d610c5be6d

    substr($1,length($1),1) -- 获取第一层目录的名字
    substr($1,length($1)-2,2) -- 获取第二层目录名字

还有一种方式是遍历文件内容删除的：

    find /path/to/cache -type f -exec grep -l "KEY: http://hostname/path/to/test.html/" {} \; | xargs rm -f
    遍历缓存目录下的所有文件，查找文件内容包含"KEY: http://hostname/path/to/test.html/"进行删除。
KEY后面的内容就是proxy_cache_key配置的实际内容，而不是md5值。

### 1.6缓存命中率
计算缓存命中率需要根据nginx的access.log进行计算，log格式配置upstream_addr, 如果是命中，upstream_addr的值为`-`。

### 1.7 Cache Loader和Cache Manager
如果启用了proxy cache，nginx还会启动Cache Loader和Cache Manager进程。

* Cache Loader： nginx启动后，需要把文件缓存元数据读进内存中，简而言之就是把每个文件的key一个个的读进内存。当这些操作完成后，Cache Loader的使命就结束了，进程也随之关掉。
* Cache Manager： 周期性的检查缓存的目录的大小是否超过了`max_size`,如果超过了，那么把最久未访问的文件从磁盘中进行删除。


### 2 注意事项
nginx采用了异步、事件驱动的方法来处理连接。这种处理方式无需（像使用传统架构的服务器一样）为每个请求创建额外的专用进程或者线程，而是在一个工作进程中处理多个连接和请求。为此，nginx工作在非阻塞的socket模式下，并使用了epoll 和 kqueue这样有效的方法。nginx从队列中取出一个事件并对其做出响应，比如读写socket。在多数情况下，这种方式是非常快的（也许只需要几个CPU周期，将一些数据复制到内存中），nginx可以在一瞬间处理掉队列中的所有事件。

但是，如果nginx要处理的操作是一些又长又重的操作，又会发生什么呢？整个事件处理循环将会卡住，直到这个事件操作执行完毕，下个事件才能被处理。

而在启用proxy cache后，这种又长又重的活很可能就是读写磁盘。在FreeBSD系统中，提供了异步读写文件的接口，可以配置nginx使用，nginx直接管理文件描述符；在Linux中，虽然提供了异步接口，但是需要设置O_DIRECT，这将导致任何对文件的访问都忽略磁盘缓存，直接从磁盘中获取，这增加磁盘的负载。在用`dd` 来测试磁盘的IO，设置O_DIRECT后，测试数据下降的厉害。

对于这种情况，nginx提出了线程池进行解决。

#### 2.1 线程池
当工作进程需要执行一个潜在的长操作时，工作进程不再自己执行这个操作，而是将任务放到线程池队列中，任何空闲的线程都可以从队列中获取并执行这个任务。

线程池的配置非常简单、灵活。我们只需要在http、 server，或者location上下文中包含aio threads指令即可：

    thread_pool default threads=32 max_queue=65536;
    aio threads=default;

* default： 线程池的名字；
* theads=32： 线程池线程数量；
* max_queue： 任务队列最多支持65536个请求；

在proxy cache中配置为：

    server {
            ....
            thread_pool default threads=32 max_queue=65536;
            aio threads=default;
            proxy_cache mycache;
            proxy_cache_key    $host$uri$is_args$args;

            location / {
                proxy_pass http://origin:80;
            }
        }

上面配置使用线程池来处理磁盘的读写操作。

#### 2.2 磁盘页缓存
虽然上面介绍了使用线程池优化读写磁盘，但是也不用着急去使用。因为nginx在缓存的时候会主动发一些hint给系统，系统把相关的页进行缓存；就算没有nginx的主动提示，Linux中也会把频繁访问的页缓存下来。这样nginx从磁盘缓存中读取数据也是非常快的。

所以如果缓存的内容少，没必要使用线程池，毕竟把读写放到连接池中也是需要开销的。

### 总结
为了获得更好的性能，缓存目录通常配置为单独的磁盘，有条件可以使用性能更好的SSD，nginx对静态文件的读取进行了优化，从文件缓存中读取比请求后端服务器读取，速度快了一个数量级；所以在频繁访问的文件使用文件缓存性能理想。