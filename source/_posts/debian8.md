---
title: debian8之systemd
tags: 技术
categories: 未分类
date: 2017-02-07 10:34:15
type:
---

## debian8之systemd
2015年4月25日，debian项目组宣布了debian8的正式稳定版本，代号为**Jessie**，该版本将在接下来的5年内获得支持。Jessie一个重大的特性就是之前倍受争议的systemd成为了默认的init系统，而sysv init在此版本依然可用，以实现版本的平滑过度。

### 1 systemd新特性
systemd能取代sysv init系统，它有几大特点：

* 支持并行化启动服务：
sysinit启动服务是一项一项依序启动的，因此没有依赖关系的服务也是要一个个的等待的。systemd就是可以让服务同时并发启动，同时采用socket式与D-Bus总线式激活服务，因此你会发现系統启动的速度变快了。

* 按需启动守护进程（daemon）：
当sysv init系统初始化时，它会将所有可能用到的后台服务进程全部启动运行，并且系统必须等待所有的服务都启动就绪后，才允许用户登录。这种做法有两个缺点：首先是启动时间过长；其次是浪费了系统资源。某些服务很可能在很长一段时间内，甚至整个服务器运行期间都没有被使用过。花费在启动这些服务上的时间是不必要的；同样，花费在这些服务上的系统资源也是浪费的。systemd可以提供按需启动能力，只有在某个服务被真正请求时才启动它。当该服务结束后，systemd可以关闭它，等待下次需要时再次启动它。

* 利用cgroups监视进程：
用cgroups代替PID来追踪进程，以此即使是两次fork之后生成的守护进程也不会脱离systemd的控制。当进程创建子进程时，子进程会继承父进程的Cgroup。因此无论服务如何启动新的子进程，所有的这些相关进程都会属于同一个Cgroup，systemd只需要简单地遍历指定的Cgroup，即可正确地找到所有的相关进程，将它们一一停止。

* 支持快照和系统恢复：
Systemd 快照提供了一种将当前系统运行状态保存并恢复的能力。
比如系统当前正运行服务 A 和 B，可以用 systemd 命令行对当前系统运行状况创建快照。然后将进程 A 停止，或者做其他的任意的对系统的改变，比如启动新的进程 C。在这些改变之后，运行 systemd 的快照恢复命令，就可立即将系统恢复到快照时刻的状态，即只有服务 A，B 在运行。一个可能的应用场景是调试：比如服务器出现一些异常，为了调试用户将当前状态保存为快照，然后可以进行任意的操作，比如停止服务等等。等调试结束，恢复快照即可。

* 维护挂载点和自动挂载点： systemd 监视所有的挂载点的进出情况，也可以用来挂载或卸载挂载点。

* 各服务间基于依赖关系进行精密控制：
由于systemd可以进行定义服务依赖性，因此如果B服务是依赖A服务的，那当你没有启动A服务的情况下手动启动B服务， systemd会自动帮你启动A服务。这样免去管理员一项一项分析服务的麻烦。

* 提供日志服务：
systemd提供了自己日志系统称为 journal。 使用 systemd 日志，无需额外安装日志服务（syslog）。systemd journal 用二进制格式保存所有日志信息，用户使用 journalctl 命令来查看日志信息，无需自己编写复杂脆弱的字符串分析处理程序。

* 和system init向后兼容：
systemd 提供了和 sysvinit 以及 LSB initscripts 兼容的特性。系统中已经存在的服务和进程无需修改也能正常运行。这降低了系统向 systemd 迁移的成本，使得 systemd 替换现有初始化系统成为可能。

如何知道自己的init系统是否是systemd？

* init系统作为第一个进程启动，指定PID为1，/proc文件系统可以获得程序的启动路径。

    ls -al /proc/1/exe
    rwxrwxrwx 1 root root 0 Jan 10 00:25 /proc/1/exe -> /lib/systemd/systemd

* 运行`ls -al /sbin/init`

    ls -al /sbin/init
    rwxrwxrwx 1 root root 20 Apr 16  2015 /sbin/init -> /lib/systemd/systemd

从上面看到init系统是systemd。


### 2 systemd约定
systemd将以前的服务启动脚本都当成**unit**单元，每种服务依据功能来分，可以分为不同的类型。可以认为一个服务是一个配置单元；一个挂载点是一个配置单元；一个交换分区的配置是一个配置单元。 基本的类型有：系统服务（.service）、挂载点（.mount）、sockets（.sockets ）、系统设备、交换分区/文件(.device)、启动目标（.target）、文件系统路径（.path）、由systemd管理的计时器(.timer)。详情参阅 man 5 systemd.unit。unit文件统一了过去各种不同的系统资源配置格式，通过文件的后缀名来区分这些配置文件。这些unit文件通常放在下面几个目录中。

* /usr/lib/systemd/system/：安装软件时主要使用的保存目录，重新安装软件会覆盖。有点类似以前的 /etc/init.d 底下的启动脚本；
* /run/systemd/system/：系统执行过程中产生的服务unit，这些脚本优先级比/usr/lib/systemd/system/目录高！
* /etc/systemd/system/：管理员根据系统需求所建立的执行脚本，其实这个目录有点类似/etc/rc.d/rc5.d/Sxx 功能，执行优先级比 /run/systemd/system/ 高！
也就是说，系统开机启动服务是看 /etc/systemd/system/ 底下的文件，该目录下面就是-大堆的连接文件，而实际执行的脚本大都放置在 /usr/lib/systemd/system/ 底下。

unit文件按照systemd约定，应该被放置在指定的3个系统目录之一。我们实际用到大多数情况下都是service，一般都会在/etc/systemd/system目录下面创建自己的unit文件。


### 2.1 target的概念
此类 unit 为其他 unit 进行逻辑分组。它们本身实际上并不做什么，只是引用其他 unit 而已。这样便可以对 unit 做一个统一的控制。(例如：bluetooth.target 只有在蓝牙适配器可用的情况下才调用与蓝牙相关的服务，如：bluetooth 守护进程、obex 守护进程等）；
在systemd的管理体系里面，以前的运行级别（runlevel）的概念被新的运行目标（target）所取代。每次系统启动的时候都会运行与当前系统相同级别target关联的所有服务，如果服务不需要跟随系统自动启动，则完全可以忽略这个Target的内容。通常来说我们大多数的Linux用户平时使用的都是“多用户模式”这个级别，对应的target值为“multi-user.target”。

Sysvinit 运行级别 | Systemd 目标| 注释
           ------| -------- |---------
0   | runlevel0.target, poweroff.target |   关闭系统。
1, s, single |  runlevel1.target, rescue.target  | 单用户模式。
2, 4 |  runlevel2.target, runlevel4.target, multi-user.target |用户定义/域特定运行级别。默认等同于 3。
3 | runlevel3.target, multi-user.target | 多用户，非图形化。用户可以通过多个控制台或网络登录。
5   | runlevel5.target, graphical.target    | 多用户，图形化。通常为所有运行级别 3 的服务外加图形化登录。
6   | runlevel6.target, reboot.target   | 重启
emergency   | emergency.target  | 紧急 Shell


### 2.2 service unit文件
一个unit文件是长啥样子的呢，我们以最常用到的.service文件来说明，以sshd.service文件为例子：

    [Unit]
    Description=OpenBSD Secure Shell server
    After=network.target auditd.service
    ConditionPathExists=!/etc/ssh/sshd_not_to_be_run

    [Service]
    EnvironmentFile=-/etc/default/ssh
    ExecStart=/usr/sbin/sshd -D $SSHD_OPTS
    ExecReload=/bin/kill -HUP $MAINPID
    KillMode=process
    Restart=on-failure

    [Install]
    WantedBy=multi-user.target
    Alias=sshd.service

可以看到unit文件就是一堆的配置项，比起之前的脚本编写是不是简易很多。一般情况下配置文件有三个段:分别是Unit,Service,Install。

1. Unit：一般写入该服务的基本信息，服务依赖关系；比如：description等等
2. Service：一般写入该服务运行的命令，运行用户，运行优先级等；
3. Install： 服务安装信息，它不在 systemd 的运行期间使用。只在使用 systemctl enable 和 systemctl disable 命令启用/禁用服务时有用。

#### 2.2.1 Unit字段
Unit配置部分，主要包含以下字段：

* Description：一段描述这个 Unit 文件的文字。

* Documentation：指定服务的文档，可以是一个或多个文档的URL路径。

* Requires：依赖的其他 Unit 列表，列在其中的 Unit 模块会在这个服务启动的同时被启动，并且如果其中有任意一个服务启动失败，这个服务也会被终止。

* Wants：与 Requires 相似，但只是在被配置的这个 Unit 启动时，触发启动列出的每个 Unit 模块，而不去考虑这些模块启动是否成功。

* After：与 Requires 相似，但会在后面列出的所有模块全部启动完成以后，才会启动当前的服务。

* Before：与 After 相反，在启动指定的任一个模块之前，都会首先确保当前服务已经运行。

* BindsTo：与 Requires 相似，但是一种更强的关联。启动这个服务时会同时启动列出的所有模块，当有模块启动失败时终止当前服务。反之，只要列出的模块全部启动以后，也会自动启动当前服务。并且这些模块中有任意一个出现意外结束或重启，这个服务会跟着终止或重启。

* PartOf：这是一个 BindTo 作用的子集，仅在列出的任何模块失败或重启时，终止或重启当前服务，而不会随列出模块的启动而启动。

* OnFailure：当这个模块启动失败时，就自动启动列出的每个模块。

* Conflicts：与这个模块有冲突的模块，如果列出模块中有已经在运行的，这个服务就不能启动，反之亦然。

上面这些配置中，除了 Description 外，都能够被添加多次。比如前面第一个例子中的After参数在一行中使用空格分隔指定所有值，也可以像第二个例子中那样使用多个After参数，在每行参数中指定一个值。



#### 2.2.2 Service字段
这个部分的配置是最重要的，描述整个服务的核心。常用的配置项有：

* Type：服务的类型，常用的有 simple（默认类型） 和 forking。默认的 simple 类型可以适应于绝大多数的场景，因此一般可以忽略这个参数的配置。而如果服务程序启动后会通过 fork 系统调用创建子进程，然后关闭应用程序本身进程的情况，则应该将 Type 的值设置为 forking，否则 systemd 将不会跟踪子进程的行为，而认为服务已经退出。

* RemainAfterExit：值为 true 或 false（也可以写 yes 或 no），默认为 false。当配置值为 true 时，systemd 只会负责启动服务进程，之后即便服务进程退出了，systemd 仍然会认为这个服务是在运行中的。这个配置主要是提供给一些并非常驻内存，而是启动注册后立即退出然后等待消息按需启动的特殊类型服务使用

* ExecStart：这个参数是几乎每个 .service 文件都会有的，指定服务启动的主要命令，在每个配置文件中只能使用一次。

* ExecStartPre：指定在启动执行 ExecStart 的命令前的准备工作，可以有多个，所有命令会按照文件中书写的顺序依次被执行。

* ExecStartPost：指定在启动执行 ExecStart 的命令后的收尾工作，也可以有多个。

* TimeoutStartSec：启动服务时的等待的秒数，如果超过这个时间服务任然没有执行完所有的启动命令，则 systemd 会认为服务自动失败。这一配置对于使用 Docker 容器托管的应用十分重要，由于 Docker 第一次运行时可以能会需要从网络下载服务的镜像文件，因此造成比较严重的延时，容易被 systemd 误判为启动失败而杀死。通常对于这种服务，需要将 TimeoutStartSec 的值指定为 0，从而关闭超时检测，如前面的第二个例子。

* ExecStop：停止服务所需要执行的主要命令。

* ExecStopPost：指定在 ExecStop 命令执行后的收尾工作，也可以有多个。

* TimeoutStopSec：停止服务时的等待的秒数，如果超过这个时间服务仍然没有停止，systemd 会使用 SIGKILL 信号强行杀死服务的进程。

* Restart：这个值用于指定在什么情况下需要重启服务进程。常用的值有 no，on-success，on-failure，on-abnormal，on-abort 和 always。默认值为 no，即不会自动重启服务。这些不同的值分别表示了在哪些情况下，服务会被重新启动，参见下表。

服务退出原因 | no | always | on-failure | on-abnormal | on-abort | on-success
-----|----|---|----|-----|----
正常退出| | √ | | | | √
异常退出| |√  |√  |  | |
启动/停止超时| | √ | √ | √ | |
被异常KILL| | √ |√  | √ | √ |

* RestartSec：如果服务需要被重启，这个参数的值为服务被重启前的等待秒数。

* ExecReload：重新加载服务所需执行的主要命令。



#### 2.2.3 Install字段
这个段中的配置与 Unit 有几分相似，但是这部分配置需要通过 systemctl enable 命令来激活，并且可以通过 systemctl disable 命令禁用。另外这部分配置的目标模块通常是指定启动级别的.target文件，用来使得服务在系统启动时自动运行。

* WantedBy：和前面的 Wants 作用相似，被哪些模块需要到，这样做的效果是当列表中的服务启动，本服务也会启动；

* RequiredBy：和前面的 Requires 作用相似，同样后面列出的不是服务所依赖的模块，而是依赖当前服务的模块。

* Also：当这个服务被 enable/disable 时，将自动 enable/disable 后面列出的每个模块。

* Alias： 可以指定一个服务的别名，这样 systemctl command xxx.service 的时候就可以不输入完整的单元名称。

上面的sshd.service例子中使用的都是 “WantedBy=multi-user.target” 表明当系统以多用户方式（默认的运行级别）启动时，这个服务需要被自动运行。当然还需要 systemctl enable 激活这个服务以后自动运行才会生效。

### 2.3 systemd主要命令
检视和控制systemd的主要命令是systemctl。该命令可用于查看系统状态和管理系统及服务。

    systemctl list-units 列出激活的单元
    systemctl list-unit-files 查看所有已安装服务
    systemctl --failed 输出运行失败的单元

启动、结束、强制终止、重新启动和运行状态，分别对应以下几个命令。

    systemctl start <Unit名称>
    systemctl stop <Unit名称>
    systemctl kill <Unit名称>
    systemctl restart <Unit名称>
    systemctl status <Unit名称>

服务的开机自动启动的启用和取消，分别对应下面两个。

    systemctl enable <Unit名称>
    systemctl disable <Unit名称>
检查单元是否配置为自动启动：

    systemctl is-enabled <Unit名称>



## 3 systemd如何兼容sysv init
之前的启动系统通过/etc/init.d目录下面的脚本管理服务，在debian8上是如何把这些script接管过来的呢？
比如我们通过执行/etc/monit/monit启动monit服务：

    # /etc/init.d/monit stop
    [ ok ] Starting monit (via systemctl): monit.service.

服务成功启动，而且提示是通过systemctl来启动的（via systemclt）， /etc/init.d/monit status得到下面内容：

    #  /etc/init.d/monit status
    ● monit.service - LSB: service and resource monitoring daemon
       Loaded: loaded (/etc/init.d/monit)
       Active: active (running) since Sun 2016-01-10 06:02:22 PST; 4s ago
上面提示Loaded: loaded (/etc/init.d/monit)，说明我们使用的是/etc/init.d的脚本。

而我们并没有创建monit.service文件，通过执行`systemctl start monit.service`和`systemctl status monit.service`也是得到同样的结果。

我们来研究下这一神奇的过程。

我们看到monit脚本中有这么一个语句：

    . /lib/lsb/init-functions
这一行会调用`/lib/lsb/init-functions.d/40-systemd`脚本，接下来脚本的执行都是由systemd来操作。
所以，执行`/etc/init.d/monit start`实际上就是执行`sysctemctl start monit.service`
但是monit.server文件并不存在，该怎么办呢？

当一个需要启动一个service时，如果找不到unit配置文件，systemd会查找一个同名的sysV init脚本，然后动态的创建一个service unit的文件。创建的依据就是该启动脚本，systemd解析标准的LSB脚本头部注释，然后组装一个unit，最后通过systemctl运行。

LSB脚本标准头部注释是被systemd的解析的，以monit的脚本来说明下：

    #!/bin/sh

    ### BEGIN INIT INFO
    # Provides:          monit
    # Required-Start:    $remote_fs
    # Required-Stop:     $remote_fs
    # Should-Start:      $all
    # Should-Stop:       $all
    # Default-Start:     2 3 4 5
    # Default-Stop:      0 1 6
    # Short-Description: service and resource monitoring daemon
    # Description:       monit is a utility for managing and monitoring
    #                    processes, programs, files, directories and filesystems
    #                    on a Unix system. Monit conducts automatic maintenance
    #                    and repair and can execute meaningful causal actions
    #                    in error situations.
    ### END INIT INFO

其脚本描述信息用### BEGIN INIT INFO 和 ### INIT INFO来分隔，systemd完全解析这里面的信息，Required-Start关键字来申明在运行该脚本之前应该需要先运行哪些脚本；
类似的Required-Stop里定义的启动服务应该在先停止哪些脚本；
Should-Start关键字和Should-Stop关键字的概念和Required-Start及Required-Stop类似,只是对其后面定义的行为是希望而不是必须；
Default-Start和Default-Stop 定义了缺省情况下该脚本在哪些运行级别下启动和停止。

systemd启动monit时调用`/lib/systemd/system-generators/systemd-sysv-generator`，解析init脚本，并动态组装一个monit.service文件，放置在`/run/systemd/generator.late/`目录下面。

在`/run/systemd/generator.late`目录下，我们找到生成的monit.service文件，内容如下：

    # Automatically generated by systemd-sysv-generator

    [Unit]
    SourcePath=/etc/init.d/monit
    Description=LSB: service and resource monitoring daemon
    Before=runlevel2.target runlevel3.target runlevel4.target runlevel5.target shutdown.target
    After=remote-fs.target all.target
    Conflicts=shutdown.target

    [Service]
    Type=forking
    Restart=no
    TimeoutSec=5min
    IgnoreSIGPIPE=no
    KillMode=process
    GuessMainPID=no
    RemainAfterExit=yes
    SysVStartPriority=4
    ExecStart=/etc/init.d/monit start
    ExecStop=/etc/init.d/monit stop
    ExecReload=/etc/init.d/monit reload

这个文件第一行就告诉我们是由systemd-sysv-generator自动生成的。

如果不想monit脚本定向到systemd，可以通过设置环境变量

    export _SYSTEMCTL_SKIP_REDIRECT=1
这样如果运行/etc/init.d/monit start, 就脱离systemd的管理了。

systemd也不是100%与sysv兼容的，比如systemctl只提供有限的操作，也不提供参数支持，而init脚本可以多样性定制；systemctl也不支持交互式启动服务等，更多不兼容可以查看[Compatibility with SysV](http://www.freedesktop.org/wiki/Software/systemd/Incompatibilities/)。

## 4编写一个service文件
有了前面的了解，写一个unit服务文件，小菜一碟啦。
以写nginx.service为例子，并加入到开机启动流程中。
第一步在/etc/systemd/system目录下面，新建一个nginx.service文件。

### 4.1 配置Unit字段
    [Unit]
    Description=The NGINX HTTP and reverse proxy server
    After=network.target remote-fs.target nss-lookup.target

这里我们只需配置两项，Description和After，After表示nginx需要在网络相关服务、远程文件系统挂载点服务、主机和网络名字查找服务这些服务启动后，本服务才执行启动；


### 4.2 编写Service字段配置

    [Service]
    Type=forking
    PIDFile=/run/nginx.pid
    ExecStartPre=/usr/sbin/nginx -t
    ExecStart=/usr/sbin/nginx
    ExecReload=/bin/kill -s HUP $MAINPID
    ExecStop=/bin/kill -s QUIT $MAINPID
    Restart=on-failure
    User=nginx
    Group=nginx

设置Type=forking，是因为nginx是父进程调用fork()创建子进程，然后父进程退出，子进程继续服务的形式；
同时设置 PIDFile= 选项，以帮助 systemd 准确定位该服务的主进程，进而加快后继单元的启动速度。
ExecStartPre：配置在启动进程前运行`/usr/sbin/nginx -t`来检查下配置语法是否正确；
ExecStart：启动nginx命令；
ExecReload：reload命令，这里是用发送信号的方式，也可以是`/usr/sbin/nginx -s reload`命令；
ExecStop：停止命令，这里也是处理信号的方式，同样可用`/usr/sbin/nginx -s stop`命令；
Restart：on-failure表示在异常退出时重启服务；
User、Group：指定运行服务用户和用户组；

### 4.3 配置Install字段
这部分是安装的相关信息，运行systemctl enable时用到。

    [Install]
    WantedBy=multi-user.target
上面的配置表示nginx服务安装在多用户模式下，当系统运行在多用户模式时，nginx服务启动。

###4.4 加载服务
当我们增加或者修改一个unit文件时，需要重新加载告知systemd。

    systemctl daemon-reload

执行这步后就可以用systemctl操作服务了。

    systemctl start nginx.service   ##启动服务
    systemctl status nginx.service  ##查看服务状态
    journal -b -u nginx.service     ##查看当次启动服务日志

使用`systemctl enable nginx.service`命令实现开机启动。

## 总结
systemd是一个先进的init系统，统一了系统各资源启动管理，有一种兼容并包，大统一体，化繁为简的哲学思想，值得大家用起来。

## 参考文献：

<http://www.nljb.net/default/Linux%E5%B7%A8%E5%A4%A7%E5%8F%98%E9%9D%A9%E4%B9%8BSystemd%E5%8F%96%E4%BB%A3SysV%E7%9A%84Init/>
<http://www.ibm.com/developerworks/cn/linux/1407_liuming_init3/>
<http://www.freedesktop.org/wiki/Software/systemd/Incompatibilities/>
<http://blog.csdn.net/institute/article/details/20152317>