---
title: Ceph手动部署
tags: ceph
categories: 未分类
date: 2017-02-09 09:51:43
type:
---

## Ceph手动部署
Ceph是一套高性能，易扩展的，无单点的、自恢复管理分布式存储系统，提供了块存储、对象存储、文件系统存储功能。Ceph强大的功能吸引越来越多的开发者使用，已有不少的公司生产环境使用它，红帽更是把Ceph的公司收购作为红帽系统的存储方案。

下面是我手动部署ceph的过程，希望对大家有所借鉴。

#### 编译ceph，构建本地的apt源
1. 下载源码，构建deb包

```
        git clone https://github.com/ceph/ceph.git
        cd ceph
        ./install-deps.sh
        apt-get install dpkg-dev
        dpkg-checkbuilddeps        # make sure we have all dependencies
        dpkg-buildpackage -j32 -uc -us

```

**-j32表示并发编译数，一般为核数的两倍即可，加快编译速度。**
**-us -uc表示不需要进行签名**
**注意编译的机器的内存最好有4G上，不然编译过程中会报错**

2. 编译成功后，在构建成功的deb包存放在ceph的上一层目录。

```
        ceph_10.2.0-1.dsc
        ceph_10.2.0-1.tar.gz
        ceph-base_10.2.0-1_amd64.deb
        ......
```

3. 为apt构建本地源。

（1）创建一个目录，保存ceph的deb包

```
        mkdir /home/ceph/debs

```

（2） 拷贝deb包到debs目录
（3）使用dpkg-scanpackages 命令生成APT可以使用的软件包索引文件

```
    dpkg-scanpackages debs /dev/null | gzip > packages/Packages.gz
```

注意一下路径问题。等待系统扫描完所有的软件包后，会返回命令行，并且在packages文件夹中生成一个名为Packages.gz的压缩文件，存有这个文件夹中的软件包信息及其依赖关系。
(4) 修改/etc/apt/sources.list来使用本地源，添加

```
    deb file:///home/ceph debs/
```

**ps: 注意斜杠和空格**

(5)运行更新：

```
    apt-get update
```

之后，安装ceph就会选择本地源了。


#### 或者使用debian 8 已编译好的源

```
    deb http://download.ceph.com/debian-jewel/  jessie main
```

上面添加的是ceph的jewel版本官方源，可以使用国内的源

```
    deb http://mirrors.163.com/ceph/debian-jewel/  jessie main
```

需要添加更新public key：

```
    curl https://git.ceph.com/release.asc | sudo apt-key add -
    apt-get update
```

### 开通防火墙
打开6789和6800-7300的端口

### 设置hosts

```
    10.160.88.220 ceph-node1
    10.160.88.221 ceph-node2
    10.160.88.222 ceph-node3
```

同时修改每个主机hostname， 使用`hostname -s`命令查看修改是否生效。
### 创建ceph用户

```
    echo "ceph ALL = (root) NOPASSWD:ALL" | tee /etc/sudoers.d/ceph
    chmod 0440 /etc/sudoers.d/ceph
```

### 部署monitor

#### 1. 生成uuid，作为集群的唯一标识

使用uuidgen生成唯一的id

```
    apt-get install uuid-runtime
    uuidgen
```

#### 2. 初步配置集群

```
    [global]
    fsid = cff8e3d5-abcc-4d6a-aca8-f29d68cdac56
    cluster = ceph
    auth cluster required = cephx
    auth service required = cephx
    auth client required = cephx

    public network = 106.2.71.0/24
    cluster network = 10.160.88.0/24

    mon initial members = ceph-node1
    mon host = 10.160.88.220
```

#### 3.创建密钥环

```
    #为monitor生成密钥
    ceph-authtool --create-keyring /tmp/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'
    #生成 `client.admin` 用户并添加该用户到密钥环
    ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --set-uid=0 --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow'
    #添加 `client.admin` 密钥到 `ceph.mon.keyring`
    ceph-authtool /tmp/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring
```

#### 4. 建立monitor map
使用主机短名、IP地址和fsid建立mornitor map

```
    monmaptool --create --add {hostname} {ip-address} --fsid {uuid} /tmp/monmap
    monmaptool --create --add node1 10.160.88.220 --fsid cff8e3d5-abcc-4d6a-aca8-f29d68cdac56 /tmp/monmap
```

#### 5. 建立monitor数据存放目录

```
    mkdir -p /var/lib/ceph/mon/{cluster-name}-{hostname}
    mkdir -p /var/lib/ceph/mon/ceph-node1
```

#### 6. 初始化mon节点

```
    ceph-mon --mkfs -i {hostname} --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring
    ceph-mon --mkfs -i node1 --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring
```

#### 7. 完成monior节点初始化

```
    touch /var/lib/ceph/mon/ceph-node1/done
```

#### 8. 启动monitor节点
对于有些系统，如debian 8 jessie用sysvinit，需要`touch /var/lib/ceph/mon/ceph-node1/sysvinit` 否则在    /etc/init.d/ceph脚本的check_host处由于找不到hostname直接退出。

**注意还需要将相关的目录所有者更改为ceph**

```
    chown ceph:ceph /var/lib/ceph -R

    /etc/init.d/ceph start mon.{hostname}
    /etc/init.d/ceph start mon.node1
```

#### 9. 检查monitor状态和集群状态

```
    # ceph mon_status
    {"name":"node1","rank":0,"state":"leader","election_epoch":3,"quorum":[0],"outside_quorum":[],"extra_probe_peers":[],"sync_provider":[],"monmap":{"epoch":1,"fsid":"cff8e3d5-abcc-4d6a-aca8-f29d68cdac56","modified":"2016-05-06 17:47:01.476949","created":"2016-05-06 17:47:01.476949","mons":[{"rank":0,"name":"node1","addr":"10.160.88.220:6789\/0"}]}}

    # ceph -s
    cluster cff8e3d5-abcc-4d6a-aca8-f29d68cdac56
     health HEALTH_ERR
            64 pgs are stuck inactive for more than 300 seconds
            64 pgs stuck inactive
            no osds
     monmap e1: 1 mons at {node1=10.160.88.220:6789/0}
            election epoch 3, quorum 0 node1
     osdmap e1: 0 osds: 0 up, 0 in
            flags sortbitwise
      pgmap v2: 64 pgs, 1 pools, 0 bytes data, 0 objects
            0 kB used, 0 kB / 0 kB avail
                  64 creating
```

### 增加OSD
#### 生成osd的序号

```
    ceph osd create [{uuid}]
```

uuid可以不要，ceph自动生成一个
#### 磁盘分区

```
    fdisk /dev/sdb
    输入n，增加一个分区
    Partition type选择P，表示主分区
    Partition number选择默认的
    First sector：选择默认
    Last sector：根据需要输入大小（如100G, 则100*1024*1024*1024/sector_size - First_secor）
    输入：w   进行写入
```

#### 创建默认目录

```
    mkdir -p /var/lib/ceph/osd/ceph-{osd-number}
    mkdir -p /var/lib/ceph/osd/ceph-2
```

#### 格式化磁盘，挂载目录

```
    mkfs -t {fstype} /dev/{hdd}
    mount -o user_xattr /dev/{hdd} /var/lib/ceph/osd/ceph-{osd-number}
    如：
    mkfs -t ext4 /dev/sdb1
    mount -o user_xattr /dev/sdb1 /var/lib/ceph/osd/ceph-2
```

#### 初始化OSD的数据目录

```
    ceph-osd -i {osd-num} --mkfs --mkkey --osd-uuid [{uuid}]
    如：
    ceph-osd -i 2 --mkfs --mkkey
```

在运行 ceph-osd --mkkey 之前，该目录必须是空的。并且，ceph-osd 工具需要用参数 --cluster 指定自定义的集群名。

#### 注册OSD的认证密钥

```
    ceph auth add osd.{osd-num} osd 'allow *' mon 'allow profile osd' -i /var/lib/ceph/osd/ceph-{osd-num}/keyring
    如：
    ceph auth add osd.2 osd 'allow *' mon 'allow profile osd' -i /var/lib/ceph/osd/ceph-2/keyring
```

#### 添加Ceph节点到CRUSH映射中
如果CRUSH映射为空，则应创建节点，下面是创建一个类型为host的节点：

```
    ceph osd crush add-bucket {hostname} host
```

#### 移动节点到root节点下面

```
    ceph osd crush move node1 root=default
```

#### 添加OSD节点到CRUSH映射中
把OSD添加到CRUSH映射，这样就可以开始接受数据了。 同样也可以反编译CRUSH的映射，把OSD添加到设备列表，把主机作为一个bucket(在它还没有加入CRUSH映射前），在主机中把设备当做项目添加，指定一个权重后重新编译并设置。

```
    ceph osd crush add {id-or-name} {weight} [{bucket-type}={bucket-name} ...]
    例如：
    ceph osd crush add osd.2 0.10 host=node1
    将osd.2加入到node1下面。
```

#### 添加OSD配置到ceph.conf中
如下面的配置[osd]是全局配置， [osd.2]是针对某个osd进行配置，会覆盖[osd]的配置

```
    [osd]
    osd journal size = 1024
    filestore xattr use omap = true
    osd max object name len = 256
    osd max object namespace len = 64
    osd_op_threads = 16
    osd_disk_threads = 4
    filestore op threads = 32

    [osd.2]
    host = node3
```

### 启动osd进程
sysvint下需要创建如下的空文件：

```
    touch /var/lib/ceph/osd/{cluster-name}-{osd-num}/sysvinit
    如：
    touch /var/lib/ceph/osd/ceph-2/sysvinit
```

还需要修改ceph目录的所有者为ceph

```
    chown ceph:ceph /var/lib/ceph/osd/ceph-2 -R
启动进程：
    /etc/init.d/ceph start osd.2
```


### 参考文档：
http://www.quts.me/2015/03/02/ceph-deploy/#long-form
http://cw760.github.io/2014/12/22/ceph%E9%9B%86%E7%BE%A4%E6%90%AD%E5%BB%BA%E6%89%8B%E8%AE%B0/
https://www.zybuluo.com/gump88/note/440777
