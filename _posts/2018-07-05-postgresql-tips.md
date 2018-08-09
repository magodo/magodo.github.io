---
layout: "post"
title: "postgresql学习笔记 (v9.6)"
categories: "blog"
tags: ['db']
published: true
comments: true
script: [post.js]
excerpted: |
---

* TOC
{:toc}

# 1 备份与恢复

PG支持三种类型的备份方式：

1. SQL dump

    - **描述**：SQL备份。使用`pg_dump`备份某个database（不包括cluster-level的对象，例如: role, tablespace），或者使用`pg_dumpall`备份整个cluster。

    - **优点**
    
        1. 可以在线备份
        2. 占用空间小

    - **缺点**

        1. 使用`pg_dumpall`备份整个cluster的时候，由于内部对每个database分别使用`pg_dump`，虽然保证了每个database内部状态是一致的，但是无法做到database之间（也即整个cluster）的状态是同步的。
        2. 速度慢

2. 文件系统级别的备份

    - **描述**：物理备份。使用支持*consistent snapshot*的文件系统作为`PGDATA`目录的挂载介质，在需要备份的时候对该文件系统做一个`forzen snapshot`。之后在需要恢复备份的时候，将这个snapshot恢复，然后启动服务。由于snapshot保存了所有文件当时的状态，PG会认为刚经历过一次crash，因此会利用WAL进行恢复（这是正常行为）。

    - **优点**

        1. 可以在线备份
        2. 速度快

    - **缺点**

        1. 占用空间大

3. continuous archiving

    - **描述**: `basebackup + WAL archive` 需要先创建一个basebackup（并非一致性的备份），然后利用归档的WAL进行replay，最终恢复一个一致的cluster。这里需要备份的是WAL archive.

    - **优点**

        1. 不需要创建一个一致性的备份用于恢复，因为那些不一致的地方可以通过后续的WAL追加可以达到一致
        2. 可以通过持续备份WAL来做到持续备份整个数据库
        3. 支持PIT(point-in-time)恢复，允许恢复到某个WAL对应的状态
        4. 允许将WAL复制到另一台机器上，从而创建一个从库

    - **缺点**
        
        1. 无法备份配置文件（因为它们不是通过SQL去修改的，而是手动修改的）
        2. 需要对每一次WAL archive的失败进行快速的修复，否则如果服务器一旦毁坏，那么从失败点开始的数据都无法恢复

以下重点介绍第三种备份的配置。

## 1.1 基于continuous archiving的备份

由于WAL默认每16MB（编译时确定）保存一份，以数字命名文件名，代表其在WAL sequence上的抽象位置点。_如果不设置`archive_mode`，那么PG仅仅会保留一部分感兴趣的WAL文件，那些位置点早于checkpoint的WAL文件则会被回收_(需要文献确认...)。而如果打开了`archive_mode`，那么每一份新产生的WAL文件都会被输入到`archive_command`中，由该命令来备份WAL。

### 1.1.1 配置WAL archiving

需要在*postgresql.conf*中做以下设置：

1. `wal_level`为至少`replica`
2. `archive_mode`为至少`on`
3. `archive_command` (返回0代表成功归档一个WAL日志，可以运行时修改，需要reload配置。当PG接收到0的返回值以后，它会认为该WAL文件已被正确的备份，因此会删除该WAL文件；否则它会重试)

### 1.1.2 创建base backup

有两种方式可以创建base backup：

1. 使用`pg_basebackup`工具
2. 使用non-exclusive lowlevel API

#### 1.1.2.1 pg_basebackup创建base backup

#### 1.1.2.2 non-exclusive lowlevel API创建base backup

1. 在目标机上使用superuser或者别的有EXECUTE权限的用户连接到源库执行：`SELECT pg_start_backup('label', false, false)`，这个连接必须一直保持直到备份结束(会执行一次checkpoint)。

    `pg_start_backup()`: 第一个参数可以取一个唯一的名字代表这次备份的标识；第二个参数代表是否限定备份使用的I/O量以减少其对其他Query操作的影响(见`checkpoint_completion_target`)；第三个参数代表是否是`exclusive`，目前`exclusive`已经被弃用，所以总是false。

2. 使用命令备份文件目录(cp, tar, rsync, etc.). 注意：以下源库上的文件不应该备份：

    - _pg_replslot/*_
    - _pg_xlog/*_
    - *postmaster.pid*
    - *postmaster.opts*

3. 在与之前start相同的连接中执行：`SELECT * FROM pg_stop_backup(false)`，这会退出备份模式，在primary上，它还会自动切换下一个WAL segment;如果是在standby上做备份，需要在primary上手动执行`pg_switch_xlog`，从而保证恢复需要的WAL log都被archive了。这个操作是为了保证备份过程中的最后一个WAL log被archive.

    `pg_stop_backup()`：的第一个返回值是恢复备份需要的最后一个WAL 位置（也即备份过程中生成的最后一个WAL）的文件名；第二个返回值需要写到一个名为`backup_label`的文件，位于目标库的PGDATA目录；第三个返回值如果非空的话要写入目标库的PGDATA目录下的名为`tablespace_map`的文件中。

4. 至此，一个完整的备份已经完成。`pg_stop_backup`的第一个返回值即是恢复备份需要的最后一个WAL segment的名字。

## 1.2 Point-in-time 恢复

基本步骤如下：

1. （如果数据库正在运行）停止数据库
2. （如果有足够的空间）将数据目录和表空间另存一份
3. 删除数据目录和表空间
4. 从base backup恢复数据目录和表空间（注意权限和所有权）。如果使用了表空间，则确保*pg_tblspc/*中的软链接被正确的恢复了
5. 删除*pg_xlog/*目录中的内容（因为这里的WAL文件是在创建basebackup时刻的内容，备份开始以后的WAL都通过`archive_command`被归档了）
6. > If you have unarchived WAL segment files that you saved in step 2, copy them into pg_xlog/. (It is best to copy them, not move them, so you still have the unmodified files if a problem occurs and you have to start over.)
7. 在数据目录下创建一个*recovery.conf*文件，并且填写相关内容（例如:`restore_command`, `stopping_point`）。同时，修改*pg_hba.conf*以阻止用户连接这个数据库
8. 启动服务器。此时，服务器会进入恢复模式，并且会通过`restore_command`读取archived的WAL log并且应用它们。如果中途出现任何错误，可以重启服务，重启后它会继续恢复。直到全部的WAL日志被应用完毕，它会将*recovery.conf*重命名为*recovery.done*，并且开始正常的数据库操作。
9. 检查数据库是否已经恢复，如果是，则修改*pg_hba.conf*恢复用户的访问权

如果仅仅需要的是从备份恢复(而非PIT恢复)，那么执行到第四步即可。


## 1.3 timeline

假设你在周四晚上5:15PM误删了一个库，但是当时并没有意识到，直到周日早上才发现。此时，你可能会拿出备份恢复到周四晚上5:14PM的状态。之后，你意识到实际上那个被误删的库其实并不重要，而从周四到周日的数据的丢失更严重。此时，如果你想再恢复到周日早上的状态，这是做不到的。这是因为数据库会将原本周四5:15PM以后的WAL日志修改成你之前恢复的那个点以后发生的事件。

幸运的是，PG通过**timeline**的概念解决了这个问题。在PG中，每当一个恢复操作成功就会新产生一个timeline，这个timeline以ID的形式作为WAL日志文件名的一部分，仅记录该次恢复之后产生的WAL日志。同时，PG会创建*timeline history*文件，用于记录这个恢复是属于哪个timeline以及恢复到何时的。

示意图：

![timeline](/assets/img/postgresql/timeline.png)

## 1.4 实践

[基于Docker的PITR实践](https://github.com/magodo/docker_practice/tree/master/postgresql)

# 2 基于Log-shipping的高可用

*这章的内容基于PG9.6[官网](https://www.postgresql.org/docs/9.6/static/warm-standby.html)*

除了基于log-shipping的高可用方案外，还有一些别的方案，可以参加官网的[这个](https://www.postgresql.org/docs/9.6/static/different-replication-solutions.html)页面。

对于基于Log-shipping的高可用方案，还有如下几个维度可以细分：

1. 复制方式：

    PG支持*日志归档(file based)*和*流式复制(record based or streaming)*。

    - 日志归档: 每次会传输一个WAL log segment(16MB, 编译期决定), 在*recovery.conf*的`restore_command`中定义如何读取一个WAL log segment, 
    - 流式复制: 通过TCP连接接收WAL log
        - 异步（默认）
        - 同步

2. standby的类型：

    standby可以是warm standby，仅用于与primary保持一致，作为备库使用；也可以是hot standby，在与primary保持一致的基础上对外提供读的功能。

需要注意的是，log shipping是异步的过程，也即当primary宕机的时候正在记录的WAL可能还没有同步到standby上，因此会造成standby上的数据丢失。这个数据丢失的时间窗口对于不同WAL传输方式可以做调整：

1. 对于日志归档：可以调节 `archive_time` 来缩短做WAL archive的周期（但是这会增加IO带宽的消耗）
2. 对于流式复制：其窗口会小的多

## 2.1 前提条件

对于运行primary机器和standby机器，这种高可用方案有以下的条件：

1. 系统架构应该保持一致(32-bit or 64-bit)
2. (如果使用了表空间)表空间相关的路径应该保持一致
3. PG版本相同

## 2.2 standby的行为

![standby server](/assets/img/postgresql/standby.png)

如上图所示，一个DB实例有一个模式叫做**Standby Mode**，在这个模式下它会不断地去读WAL，然后应用到自身。

而退出这个模式的条件有三个：

- DB被停止 -> stop mode
- `pg_ctl promote`在standby上被执行 -> ?
- trigger file在standby上被创建 -> ?

## 2.3 配置primary

### 2.3.1 基于文件的log-shipping

配置*continuous archiving*，归档的目标地址应该是一个standby也能访问的路径（即使primary宕机）.

### 2.3.2 基于流的log-shipping

1. 创建一个供standby用来复制的role
1. *pg_hba.conf*中为standby的这个role创建db为`replication`的访问项
1. *postgresql.conf*中设置`wal_level`为至少`replica`
1. *postgresql.conf*中设置`max_wal_senders`为适当值
1. *postgresql.conf*中设置`listen_address`
1. *postgresql.conf*中设置`max_replication_slots`为适当值
1. *postgresql.conf*中设置`full_page_writes`为`on`（默认即为`on`），这是运行`pg_basebackup`的前提条件
1. （如果流是基于TPC的），在 *postgresql.conf* 中设置`tcp_keepalives_idle`, `tcp_keepalives_internal`, `tcp_keepalives_count`
1. 为standby创建一个复制槽

### 2.3.3 基于以上两者

都要设置。

## 2.4 配置standby

1. 先在standby机器上从basebackup恢复

    可以使用`pg_basebackup`的如下命令：

        # rm -rf $PGDATA
        # pg_basebackup -D $PGDATA -F p -R -S [slotname] -X stream [-c fast] -d [connstr]

    其中`connstr`的说明可以参见[官网](https://www.postgresql.org/docs/9.6/static/libpq-connect.html#LIBPQ-CONNSTRING)，一个简单的例子如下：

        postgresql://user:password@host:port[/dbname]?application_name=foobar

    `connstr`相对于在命令行上以`-h`,`-U`等来指定连接参数的好处：

    - 它可以指定密码，否则的话你需要准备一个*~/.pgpass*文件来存放密码
    - 它允许加入额外的连接参数（例如：`application_name`），而这些额外的连接参数会被加入到创建出来的*recovery.conf*文件中

        

2. 新建一个*recovery.conf*文件（如果使用**pg_basebackup**恢复的话，加入`-R`参数可以自动创建该文件），在文件中做如下配置：

    1. 设置`standby_mode`为`on`
    2. （如果使用基于文件的log-shipping） 设置`restore_command`去拷贝WAL archive 
    3. （如果你要设置多个standby）设置`recovery_target_timeline`为`latest`, to make the standby server follow the timeline change that occurs at failover to another standby.
    4. （如果使用基于流的log-shipping）设置`primary_conninfo`，其内容称为**libpq connection string**，包括host name(or IP)和其他额外的选项（例如：密码）示例：`user=postgres host=170.17.0.1 port=32810 sslmode=prefer sslcompression=1 krbsrvname=postgres`
    5. （如果使用基于流的log-shipping）设置`primary_slot_name`

3. 由于这个standby在容灾后会成为primary，因此需要像之前的primary一样做基于文件或者基于流的log-shipping的配置

## 2.5 流式复制(streaming replication)

流式复制默认是异步的.

如果仅使用流式复制而没有配置日志归档(file based continuous archive)，则primary可能在standby接收到某些WAL之前将其回收。如果要避免这种情况，可以为standby配置复制槽(replication slot). 当然，如果配置了日志归档，也可以避免这种情况，因为这些被回收的日志一定会先被归档，从而使得standby可以读取并应用。

流式复制在primary和standby的基本配置在上面两节已经提过。

在standby与primary的连接被成功建立起来之后，你会在standby系统中看到多了一个**walreceiver**进程；在primary系统中看到多了一个**walsender**进程。

### 2.5.1 权限

由于WAL中包含敏感信息，因此primary应该仅允许信任的standby接收这些WAL流。而这些standby连接primary的帐号也需要有superuser权限，或者至少有`REPLICATION`权限，建议创建一个特殊的账户，仅拥有`REPLICATION`和`LOGIN`权限，用于复制，这样可以避免standby修改primary的数据。因此，正确的步骤如下：

1. 在primary上为standby创建一个复制帐号`foo`，设置密码，并且给予其`REPLICATION`和`LOGIN`的权限:

        postgres=> CREATE USER foo WITH LOGIN REPLICATION PASSWORD '123';

2. 在primary上为`foo`开通访问权限（假设standby的IP是192.168.1.100）:

    修改*pg_hba.conf*:

        # Allow the user "foo" from host 192.168.1.100 to connect to the primary
        # as a replication standby if the user's password is correctly supplied.
        #
        # TYPE  DATABASE        USER            ADDRESS                 METHOD
        host    replication     foo             192.168.1.100/32        md5

3. 在standby上设置连接primary的连接字符串(connection string):

    修改*recovery.conf*（假设primary的IP是192.168.1.50）:

        # The standby connects to the primary that is running on host 192.168.1.50
        # and port 5432 as the user "foo" whose password is "123".
        primary_conninfo = 'host=192.168.1.50 port=5432 user=foo password=123'

    注意，这里的密码也可以放在 *~/.pgpass* 文件中。

### 2.5.2 监控

使用流式复制的时候，需要关注有多少WAL是primary中产生的，但是还没有应用到standby。这两个信息可以分别通过以下操作获得:

* 在primary上执行`pg_current_xlog_location`
* 在standby上执行`pg_last_xlog_receive_location`

想要知道详细的信息，可以通过在primary上执行`pg_stat_replication`来获得详细的统计项进一步分析，例如：其中有一项叫做`sent_location`（表示上一个通过TCP连接发送出去的WAL的位置），如果`sent_location`和`pg_current_xlog_location`差距很大，则说明primary现在负载很高；如果`sent_location`和`pg_last_xlog_receive_location`相差很大，则说明有网络延迟或者standby负载很高。

## 2.6 复制槽(replication slots)

复制槽提供以下几个作用：

- 避免primary将还未应用到standby的WAL删除
- 即使standby当前没有连接到primary，primary也不会把可能导致恢复冲突([recovery conflict](https://www.postgresql.org/docs/9.6/static/hot-standby.html#HOT-STANDBY-CONFLICT)）的行删除。

### 2.6.1 查看和操作复制槽

每一个复制槽都有一个名字，其状态可以通过`pg_replication_slots`视图来查看，同时也可以通过[流复制协议](https://www.postgresql.org/docs/9.6/static/protocol-replication.html)或者[SQL函数](https://www.postgresql.org/docs/9.6/static/functions-admin.html#FUNCTIONS-REPLICATION)来创建或删除。

### 2.6.2 简单例子

在primary上建立一个复制槽：

		postgres=# SELECT * FROM pg_create_physical_replication_slot('node_a_slot');
		  slot_name  | xlog_position
		-------------+---------------
		 node_a_slot |

		postgres=# SELECT * FROM pg_replication_slots;
		  slot_name  | slot_type | datoid | database | active | xmin | restart_lsn | confirmed_flush_lsn
		-------------+-----------+--------+----------+--------+------+-------------+---------------------
		 node_a_slot | physical  |        |          | f      |      |             |
		(1 row)

在standby上使用这个复制槽：

		standby_mode = 'on'
		primary_conninfo = 'host=192.168.1.50 port=5432 user=foo password=foopass'
		primary_slot_name = 'node_a_slot'

## 2.7 级联复制

既作为sender又作为receiver的standby称为**cascading standby**:

    upstream server -> cascading standby -> downstream server

目前的cascading standby都是异步的，所有同步的设置（在下一节讲到）在级联模式下不起作用。

## 2.8 同步流式复制

异步流式复制引入一个时间窗口，当primary宕机之后，刚产生的事务可能还没有复制给standby，导致两边状态不一致。

同步复制提供了一种手段去保证当前复制的事务被一个或多个standby所应用。这种等级的保护称为**2-safe replication**, 如果`synchronous_commit`被设为`remote_write`，则等级变为**group-1-safe**.

同步复制的缺陷在于，它增加了响应时间，大于等于primary到standby的round trip时间。

> Read only transactions and transaction rollbacks need not wait for replies from standby servers. Subtransaction commits do not wait for responses from standby servers, only top-level commits. Long running actions such as data loading or index building do not wait until the very final commit message. All two-phase commit actions require commit waits, including both prepare and commit.

### 2.8.1 基本配置

#### 2.8.1.2 配置primary

1. 按照上面流式复制配置的那一章做配置
1. 在*postgresql.conf*中设置`synchronous_commit`为`on`（`on`是默认）。值得注意的是，`synchronous_commit`可以在不同的级别被打开或者关闭，包括用户级别，DB级别，transaction级别。
1. 在*postgresql.conf*中设置`synchronous_standby_names`，格式为：

        num_sync ( standby_name[, ...] )

        or 

        standby_name [, ...]

    >  The synchronous standbys will be those whose names appear earlier in this list, and that are both currently connected and streaming data in real-time (as shown by a state of streaming in the pg_stat_replication view). Other standby servers appearing later in this list represent potential synchronous standbys. If any of the current synchronous standbys disconnects for whatever reason, it will be replaced immediately with the next-highest-priority standby. Specifying more than one standby name can allow very high availability.
    
    第一种形式中，`num_sync` 是primary需要等待的reply的从库的数目。例如：`3(s1,s2,s3,s4)` 代表这次事务需要等待s1-s4中的任意三个.

    第二种形式等价与第一种形式中的`num_sync`等于1.

    如果这个设置为空，那么同步机制不会生效。

    另外值得注意的是，这个参数是可以运行时被动态修改的。

#### 2.8.1.2 配置standby

1. 按照上面流式复制配置的那一章做配置
1. 如果*postgresql.conf*中的`wal_receiver_status_interval`为非0（默认为10 sec），那么每次commit，都可以立即在primary的`pg_stat_replication`视图中看到复制进度的更新
1. 在*recovery.conf*的`primary_conninfo`中加入`application_name`这个设置，这个会被用于primary中的`synchronous_standby_names`的设置。

### 2.8.3 性能

同步复制会降低系统性能，考虑一种情况：应用的10%的I/O是需要保证数据一致性，而90%的I/O是无关紧要的。这种情况下如果盲目地全局使用同步复制是不明智的。

由于PG支持事务级别的同步，因此针对上面的场景，建议的做法是在应用程序中做同步/异步复制的控制，即在事务中执行`SET LOCAL synchronous_commit TO on/off/remote_write/...`.

### 2.8.3 高可用

高可用的最好的解决方案是同步复制+多个从库（总是保证有potential standby），这样可以保证主库不会由于一个同步从库的宕机而被阻塞。

standby在启动后首先会进入**catchup mode**，则这个模式下standby会追primary的WAL。直到第一次完全追上primary的WAL之后，standby会进入**streaming state**。如果standby停止服务，那么它又进入**catchup mode**。仅当standby处于**streaming mode**的情况下，它才是一个同步standby，也即primary会在每一个WAL复制时等待standby的响应。

如果主库在等待ACK的过程中重启了，那么主库会认为上一次的transaction是成功的。然而，同步复制此时可以保证的是，客户端程序不会收到成功的返回。因此，当这种情况发生之后有两种处理方式：

1. 维持重启的DB为主库，那么所有同步的从库都会进入**catchup mode**，然后将刚才那条事务接收到。而客户端程序需要确认上一次的操作是否已经成功（因为前一次很可能因为DB挂掉而超时或者返回失败），如果判断已经成功，那么同样的事务不需要再做第二次。也就是说，客户端程序对这种敏感的操作总是要在写操作之后读一下来验证（如果单纯依赖于返回值，可能会导致双重操作的问题，例如对于金融应用，可能引起double payment）；

2. 重启的DB被downgrade为从库，而之前的某个从库被提升为主库，那么这个新的主库之前可能接收到了这个事务，也可能没有。此时，应用程序依然需要确认上次操作结果。

### 2.9 从库持续归档

> When continuous WAL archiving is used in a standby, there are two different scenarios: the WAL archive can be shared between the primary and the standby, or the standby can have its own WAL archive. When the standby has its own WAL archive, set archive_mode to always, and the standby will call the archive command for every WAL segment it receives, whether it's by restoring from the archive or by streaming replication. The shared archive can be handled similarly, but the archive_command must test if the file being archived exists already, and if the existing file has identical contents. This requires more care in the archive_command, as it must be careful to not overwrite an existing file with different contents, but return success if the exactly same file is archived twice. And all that must be done free of race conditions, if two servers attempt to archive the same file at the same time.

> If archive_mode is set to on, the archiver is not enabled during recovery or standby mode. If the standby server is promoted, it will start archiving after the promotion, but will not archive any WAL it did not generate itself. To get a complete series of WAL files in the archive, you must ensure that all WAL is archived, before it reaches the standby. This is inherently true with file-based log shipping, as the standby can only restore files that are found in the archive, but not if streaming replication is enabled. When a server is not in recovery mode, there is no difference between on and always modes.

 3. 服务器设置

## 3.1 WAL设置

下表列出几个重要的WAL相关的参数：

|---
|参数|类型|默认值|启动后可更改|描述
|-|-|-|-
|`wal_level`|enum| minimal|no|- minimal: 记录的信息仅能从crash或者立即关机状态下恢复数据<br>- replica: 记录的信息可以用于WAL archive或者只读从库<br>- logical: 支持逻辑解码
|`fsync`|bool|on|no|保证每一次更新动作都调用`wal_sync_method`中指定的方式写入磁盘<br>**注意**：设置为off可能会导致数据库内部数据不一致
|`synchornous_commit`|enum|on|yes|返回成功前是否等待WAL写入磁盘(如果`synchronouns_standby_names`非空，也会影响从库)<br>- off: 不会导致数据库内部数据不一致，但是可能导致客户端/从库与服务器端状态不同步<br>- on: 等待服务器以及其从库收到commit并且写入磁盘才返回<br>- remote_apply: 等待服务器以及其从库收到commit并且应用才返回（此时读操作可以看到变化）<br>- remote_write: 等待服务器以及其从库收到commit并且写入操作系统（写队列）才返回，但是并不保证写入到磁盘（这可以保证DB实例crash不会导致数据丢失；但是操作系统的crash会导致数据丢失）。不过只要主库还在，下一次从库起来以后可以继续追上。唯一可能丢失数据的情况是，主库和从库同时挂掉，并且主库的数据库无法恢复。因此，这个选项相对安全，并且比`on`更高效。<br>- local: 仅等待服务器收到commit并且写入磁盘才返回
|`wal_writer_delay`|integer|200ms|no|等多久之后将WAL写入到磁盘，在这个时间点之间的WAL仅写入操作系统的写队列，并没有写入磁盘
|`wal_writer_flush_after`|integer|1MB|no|写了多少WAL之后将WAL写入到磁盘，在这个时间点之间的WAL仅写入操作系统的写队列，并没有写入磁盘
|===

# 99. 中间件

## 99.1 pgPool2

[传送们](http://www.pgpool.net/mediawiki/index.php/Main_Page)

- session级别的连接池
- 复制，自动failover/failback 
- 负载均衡
- 达到最大连接时，新连接被加入等候队列（而不是默认的返回错误）

## 99.2 pgbouncer

[传送们](https://pgbouncer.github.io/)

支持不同级别的连接池，包括：

- session pooling
- transaction pooling
- statement pooling

## 99.3 HAProxy

[传送们](http://www.haproxy.org/)

通用的负载均衡组件，支持TCP/HTTP代理。同时，与keepalived高度集成，支持高可用。

## 99.4 Repmgr

[传送们](https://repmgr.org/)

对官方提供的复制功能的增强，支持自动failover/switchover。
