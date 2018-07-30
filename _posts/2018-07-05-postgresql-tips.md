---
layout: "post"
title: "postgresql学习笔记"
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

## 2.1 基于continuous archiving的备份

### 2.1.1 配置WAL archiving

需要在*postgresql.conf*中做以下设置：

1. `wal_level`为至少`replica`
2. `archive_mode`为至少`on`
3. `archive_command` (返回0代表成功归档一个WAL日志，可以运行时修改，需要reload配置)

### 2.1.2 创建base backup

有两种方式可以创建base backup：

1. 使用`pg_basebackup`工具
2. 使用non-exclusive lowlevel API

#### 2.1.2.1 pg_basebackup创建base backup

#### 2.1.2.2 non-exclusive lowlevel API创建base backup

1. `SELECT pg_start_backup('label', false, false)`：执行checkpoint，这个过程I/O会被限制在一定的量（基于`checkpoint_completion_target`）
2. 使用命令备份文件目录(cp, tar, rsync, etc.)
3. `SELECT * FROM pg_stop_backup(false)`：退出备份模式，在master上，它还会自动切换下一个WAL segment;如果是在standby上做备份，需要在master上手动执行`pg_switch_xlog`，从而保证恢复需要的WAL log都被archive了。这个操作是为了保证备份过程中的最后一个WAL log被archive.
4. 至此，一个完整的备份已经完成。`pg_stop_backup`的第一个返回值即是恢复备份需要的最后一个WAL segment的名字。

## 2.2 Point-in-time 恢复

基本步骤如下：

1. （如果数据库正在运行）停止数据库
2. （如果有足够的空间）将数据目录和表空间另存一份
3. 删除数据目录和表空间
4. 从base backup恢复数据目录和表空间（注意权限和所有权）。如果使用了表空间，则确保*pg_tblspc/*中的软链接被正确的恢复了
5. 删除*pg_xlog/*目录中的内容
6. > If you have unarchived WAL segment files that you saved in step 2, copy them into pg_xlog/. (It is best to copy them, not move them, so you still have the unmodified files if a problem occurs and you have to start over.)
7. 在数据目录下创建一个*recovery.conf*文件，并且填写相关内容（例如:`restore_command`, `stopping_point`）。同时，修改*pg_hba.conf*以组织用户连接这个数据库
8. 启动服务器。此时，服务器会进入恢复模式，并且会通过`restore_command`读取archived的WAL log并且应用它们。如果中途出现任何错误，可以重启服务，重启后它会继续恢复。直到全部的WAL日志被应用完毕，它会将*recovery.conf*重命名为*recovery.done*，并且开始正常的数据库操作。
9. 检查数据库是否已经恢复，如果是，则修改*pg_hba.conf*恢复用户的访问权

## 2.3 timeline

假设你在周四晚上5:15PM误删了一个库，但是当时并没有意识到，直到周日早上才发现。此时，你可能会拿出备份恢复到周四晚上5:14PM的状态。之后，你意识到实际上那个被误删的库其实并不重要，而从周四到周日的数据的丢失更严重。此时，如果你想再恢复到周日早上的状态，这是做不到的。这是因为数据库会将原本周四5:15PM以后的WAL日志修改成你之前恢复的那个点以后发生的事件。

幸运的是，PG通过**timeline**的概念解决了这个问题。在PG中，每当一个恢复操作成功就会新产生一个timeline，这个timeline以ID的形式作为WAL日志文件名的一部分，仅记录该次恢复之后产生的WAL日志。同时，PG会创建*timeline history*文件，用于记录这个恢复是属于哪个timeline以及恢复到何时的。

# 2 基于Log-shipping的高可用

*这章的内容基于PG9.6[官网](https://www.postgresql.org/docs/9.6/static/warm-standby.html)*

## 2.1 不同方案

1. 复制方式：

    PG支持*日志归档(file based)*和*流式复制(record based or streaming)*。

    - 日志归档: 每次会传输一个WAL log segment(16MB, 编译期决定), 在recoery.conf的`restore_command`中定义如何读取一个WAL log segment, 
    - 流式复制: 通过TCP连接接收WAL log
        - 异步（默认）
        - 同步

2. standby的类型：

    standby可以是warm standby，仅用于与master保持一致，作为备库使用；也可以是hot standby，在与master保持一致的基础上对外提供读的功能。


## 2.2 限制

WAL shipping是异步的过程，也即当master宕机的时候正在记录的WAL可能还没有同步到standby上，因此会造成standby上的数据丢失。这个数据丢失的时间窗口对于不同WAL传输方式可以做调整：

1. 对于日志归档：可以调节 `archive_time` 来缩短做WAL archive的周期（但是这会增加IO带宽的消耗）
2. 对于流式复制：其窗口会小的多

## 2.3 前提条件

对于运行master机器和standby机器，这种高可用方案有以下的条件：

1. 系统架构应该保持一致(32-bit or 64-bit)
2. (如果使用了表空间)表空间相关的路径应该保持一致
3. PG版本相同

## 2.4 standby的行为

![standby server](/assets/img/postgresql/standby.png)

## 2.5 配置master

配置*continuous archiving*，归档的目标地址应该是一个standby也能访问的路径（即使master宕机），例如可以设置为standby机器上的某个目录。

似乎也可以不设置archive模式，但是至少需要把`wal_level`设置为至少`replica`.

如果要支持**流式复制**，那么需要做以下几件事：

1. *pg_hba.conf*中为standby机器创建`replication`访问项
2. *postgresql.conf*中设置`max_wal_senders`为适当值
3. *postgresql.conf*中设置`listen_address`, *pg_hba.conf*中设置访问权限
4. (如果使用了replication slot)*postgresql.conf*中设置`max_replication_slots`为适当值

## 2.6 配置standby

先在standby机器上从base backup恢复。然后，新建一个*recovery.conf*文件（如果使用**pg_basebackup**恢复的话，加入`-R`参数可以自动创建该文件），在文件中做如下配置：

1. 设置`standby_mode`为`on`
2. （如果仅使用日志归档） 设置`restore_command`去拷贝WAL archive (如果仅使用流式复制，则这一步可以省略)
3. （如果你要设置多个standby）设置`recovery_target_timeline`为`latest`, to make the standby server follow the timeline change that occurs at failover to another standby.
4. （如果需要使用**流式复制**）设置`primary_conninfo`，其内容称为**libpq connection string**，包括host name(or IP)和其他额外的选项（例如：密码）示例：`user=postgres host=170.17.0.1 port=32810 sslmode=prefer sslcompression=1 krbsrvname=postgres`
5. 配置*continuous archiving*，*pg_hba.conf*（连接设置），因为这个standby在容灾后会成为master
6. （如果WAL archive仅用于standby，而不需要从备份恢复）设置`archive_cleanup_command`命令，来清理standby不需要的archive文件

## 2.7 流式复制(streaming replication)

流式复制默认是异步的，因此如果仅使用流式复制而没有配置日志归档(file based continuous archive)，则master可能在standby接收到某些WAL之前将其回收。如果要避免这种情况，可以为standby配置复制槽(replication slot). 当然，如果配置了日志归档，也可以避免这种情况，因为这些被回收的日志一定会先被归档，从而使得standby可以读取并应用。

流式复制在master和standby的基本配置在2.5节和2.6节中已经提及。

### 2.7.1 权限

由于WAL中包含敏感信息，因此master应该仅允许信任的standby接收这些WAL流。而这些standby连接master的帐号也需要有superuser权限，或者至少有`REPLICATION`权限，建议创建一个特殊的账户，仅拥有`REPLICATION`和`LOGIN`权限，用于复制，这样可以避免standby修改master的数据。因此，正确的步骤如下：

1. 在master上为standby创建一个复制帐号`foo`，设置响应的密码，并且给予其`REPLICATION`和`LOGIN`的权限:

        postgres=> CREATE USER foo WITH LOGIN REPLICATION PASSWORD '123';

2. 在master上为`foo`开通访问权限（假设standby的IP是192.168.1.100）:

    修改*pg_hba.conf*:

        # Allow the user "foo" from host 192.168.1.100 to connect to the primary
        # as a replication standby if the user's password is correctly supplied.
        #
        # TYPE  DATABASE        USER            ADDRESS                 METHOD
        host    replication     foo             192.168.1.100/32        md5

3. 在standby上设置连接master的连接字符串(connection string):

    修改*recovery.conf*（假设master的IP是192.168.1.50）:

        # The standby connects to the primary that is running on host 192.168.1.50
        # and port 5432 as the user "foo" whose password is "123".
        primary_conninfo = 'host=192.168.1.50 port=5432 user=foo password=123'

    注意，这里的密码也可以放在 *~/.pgpass* 文件中。

### 2.7.2 监控

使用流式复制的时候，需要关注有多少WAL是master中产生的，但是还没有应用到standby。这两个信息可以分别通过以下操作获得:

* 在master上执行`pg_current_xlog_location`
* 在standby上执行`pg_last_xlog_receive_location`

想要知道详细的信息，可以通过在master上执行`pg_stat_replication`来获得详细的统计项进一步分析，例如：其中有一项叫做`sent_location`（表示上一个通过TCP连接发送出去的WAL的位置），如果`sent_location`和`pg_current_xlog_location`差距很大，则说明master现在负载很高；如果`sent_location`和`pg_last_xlog_receive_location`相差很大，则说明有网络延迟或者standby负载很高。

## 2.8 复制槽(replication slots)

复制槽可以避免master将还未应用到standby的WAL删除，并且即使standby当前没有连接到master，master也不会把可能导致恢复冲突([recovery conflict](https://www.postgresql.org/docs/9.6/static/hot-standby.html#HOT-STANDBY-CONFLICT)）的行删除。

### 2.8.1 查看和操作复制槽

每一个复制槽都有一个名字，其状态可以通过`pg_replication_slots`视图来查看，同时也可以通过[流复制协议](https://www.postgresql.org/docs/9.6/static/protocol-replication.html)或者[SQL函数](https://www.postgresql.org/docs/9.6/static/functions-admin.html#FUNCTIONS-REPLICATION)来创建或删除。

### 2.8.2 简单例子

在master上建立一个复制槽：

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

## 2.9 同步流式复制

异步流式复制引入一个时间窗口，当master宕机之后，刚产生的事务可能还没有复制给standby，导致两边状态不一致。

同步复制提供了一种手段去保证当前复制的事务被一个或多个standby所应用。这种等级的保护称为**2-safe replication**, 如果`synchronous_commit`被设为`remote_write`，则等级变为**group-1-safe**.

同步复制的缺陷在于，它增加了响应时间，大于等于master到standby的round trip时间。

### 2.9.1 基本配置

在流式复制的配置基础上，master只需要在*postgresql.conf*中设置`synchronous_standby_names`，格式为：

    num_sync ( standby_name[, ...])

由于`synchronous_commit`默认为`on`，因此不需要修改。

standby在*recovery.conf*的连接字串中加入`application_name=<standby_name>`，从而使master可以知道这个standby是否为同步。

### 2.9.2 同步流式复制的过程

standby在启动后首先会进入*catchup mode*，则这个模式下standby会追master的WAL。直到第一次完全追上master的WAL之后，standby会进入*streaming state*。如果standby停止服务，那么它又进入*catchup mode*。仅当standby处于*streaming mode*的情况下，它才是一个同步standby，也即master会在每一个WAL复制时等待standby的响应。
