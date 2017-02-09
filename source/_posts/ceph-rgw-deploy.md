---
title: ceph rgw对象存储网关搭建
tags: ceph
categories: 未分类
date: 2017-02-09 10:01:28
type:
---


### ceph rgw对象存储网关搭建

ceph支持对象存储，依赖于一个网关rados gameway。此网关支持S3，Swift协议，比较常用。网关就是web服务器，ceph支持3种web服务部署。
apache、nginx以及自带的civetweb。下面我给出自己的搭建方法。

### rados gateway 搭建
#### 前置工作
1. 安装radosgw进程

```
    sudo apt-get install radosgw
```

2. 为网关配置一个密钥环：

```
        ceph-authtool --create-keyring /etc/ceph/ceph.client.radosgw.keyring
        chmod +r /etc/ceph/ceph.client.radosgw.keyring
```

3. 为每个网关示例创建一个名字和key， 比如下面创建一个名字为gateway-node1的网关：

```
        ceph-authtool /etc/ceph/ceph.client.radosgw.keyring -n client.radosgw.gateway-node1 --gen-key
```

4. 为刚创建的key分配权限

```
        ceph-authtool -n client.radosgw.gateway-node1 --cap osd 'allow rwx' --cap mon 'allow rwx' /etc/ceph/ceph.client.radosgw.keyring
```

5. 注册key到集群中

```
        ceph -k /etc/ceph/ceph.client.admin.keyring auth add client.radosgw.gateway-node1 -i /etc/ceph/ceph.client.radosgw.keyring
```

6. 把/etc/ceph/ceph.client.radosgw.keyring拷贝到radosgw的机器上。

7. 创建所需的pool，可以使用ceph的计算器http://ceph.com/pgcalc/，计算合理的值.如果不手动创建，radosgw启动进程时会自动根据配置默认的参数进行创建。

```
        ceph osd pool create .rgw.root 32 32
        ceph osd pool create .rgw.control 32 32
        ceph osd pool create .rgw.gc 32 32
        ceph osd pool create .rgw.buckets.data 256 256
        ceph osd pool create .rgw.buckets.index 128 128
        ceph osd pool create .log 64 64
        ceph osd pool create .users 64 64
        ceph osd pool create .users.email 64 64
        ceph osd pool create .users.keys 64 64
        ceph osd pool create .users.uid 64 64

```

    Ceph 对象网关有多个存储池，考虑到所有存储池会共用相同的 CRUSH 分级结构，所以 PG 数不要设置得过大，否则会影响性能。

8. 创建rgw数据目录

```
        mkdir -p /var/lib/ceph/radosgw/client.radosgw.gateway-node1
        chown ceph:ceph /var/lib/ceph -R
```

#### 使用civetweb搭建
1. 配置网关

```
        [client.radosgw.gateway-node1]
        host = node1
        rgw data = /var/lib/ceph/radosgw/client.radosgw.gateway-node1
        keyring = /etc/ceph/ceph.client.radosgw.keyring
        rgw frontends = "civetweb port=8090 enable_keep_alive=yes"
        log file = /var/log/radosgw/client.radosgw.gateway-node1.log
        rgw num rados handles = 10  ###每个线程处理几个请求
        rgw thread pool size = 500
        rgw print continue = false
        rgw cache enabled = true
        rgw content length compat = true  ##表示兼容没有content-length字段的情况
```

2. 启动rgw进程

```
        /etc/init.d/radosgw start
```

### 使用apache
1. 安装apache2

```
    apt-get install apache2
```

2. 在/etc/apache/sites-avalible下创建一个rgw.conf(文件名可改):

```
    <VirtualHost *:80>
        ServerName s3.ceph.com
        DocumentRoot /var/www/html

        ErrorLog /var/log/radosgw/rgw_error.log
        CustomLog /var/log/radosgw/rgw_access.log combined

        # LogLevel debug

        RewriteEngine On

        RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization},L]

        SetEnv proxy-nokeepalive 1

        ProxyPass / unix:///var/run/ceph/ceph.radosgw.gateway.fastcgi.sock|fcgi://localhost:9000/

    </VirtualHost>
```

3. 启用rgw.conf

```
        cd /etc/apache2/sites-enable
        ln -s ../sites-avalible/rgw.conf
```

**注意：在apache2的2.4.9版本以上才支持unix socket**

3. ceph.conf配置：

```
        [client.radosgw.gateway-node1]
        host = node1
        rgw frontends = fastcgi
        rgw data = /var/lib/ceph/radosgw/client.radosgw.gateway-node1
        keyring = /etc/ceph/ceph.client.radosgw.keyring
        rgw content length compat = true
        rgw socket path = /var/run/client.radosgw.gameway-node1.fastcgi.sock
        rgw frontends = fastcgi socket_port=9000 socket_host=0.0.0.0
        log file = /var/log/radosgw/client.radosgw.gateway-node1.log
        rgw print continue = false
        rgw num rados handles = 1000
        rgw thread pool size = 500
```

4. 启动apache进程和rgw进程

```
        /etc/init.d/apache start
        /etc/init.d/radosgw start
```

### 使用nginx
1. 安装nginx

```
    apt-get install nginx
```

2. 同样在/etc/nginx/site-avalible下新建站点rgw.conf

```
        upstream rgw {
            #server unix:/var/run/client.radosgw.gateway-node1.fastcgi.sock;
            server localhost:9000;
        }

        server {
            listen 80 default backlog=20000;

            server_name s3.ceph.com;

            client_max_body_size 0;

            proxy_buffering off;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $remote_addr;

            access_log /var/log/radosgw/access.log main;
            error_log /var/log/radosgw/error.log;

            location / {
                fastcgi_pass_header Authorization;
                fastcgi_pass_request_headers on;

                include fastcgi_params;
                fastcgi_pass rgw;
            }
        }

```

3. 启动rgw.conf站点

```
    cd /etc/nginx/site-enabled
    ln -s ../site-available/rgw.conf
```

4. ceph中rgw的配置与apache的一致。
5. 启动nginx和radosgw进程

```
    /etc/init.d/nginx start
    /etc/init.d/radosgw start
```

### 参考文档：
http://dachary.org/loic/ceph-doc/start/quick-rgw/