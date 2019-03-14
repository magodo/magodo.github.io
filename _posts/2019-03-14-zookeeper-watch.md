---
layout: "post"
title: "zookeeper 之 watch"
categories: "blog"
tags: ['distributed']
published: False
comments: true
script: [post.js]
excerpted: |
    研究下zookeeper中的watch函数与session超时之间的关系...
---

* TOC
{:toc}

# 背景

公司里面服务发现的组件用的是zookeeper，一般的过程大概是提供服务的进程会在一个约定的znode下面创建ephemeral znode (以下简称eznode)，然后内容中加入该服务的ip和port。别的服务就可以随机在这个约定的znode下挑选一个eznode，解析得到它的ip，port，向其发送tcp请求。

特别地，每个服务进程除了在启动的时候会创建(`create`)一次这个znode以外，还会定时地尝试创建。这主要是为了防止服务进程的网络和zk服务的网络分区，导致zk服务在一定的时间内（session expiration timeout）没有接收到这个服务进程的心跳从而将其定义为过期。过期会导致eznode被删除，也会导致该服务进程(client)的所有watcher被删除。

接下来，我们就结合Go语言中的`github.com/samuel/go-zookeeper/zk`库详细的分析一下会话超时和watcher相关的一些知识，并且基于这些知识讨论一下在实际应用中如何通过zk来完成一些分布式相关的任务。

# Session State Diagram

[官网](https://zookeeper.apache.org/doc/r3.4.13/zookeeperProgrammers.html)上边有对session状态迁移的示意图：

[!state](/assets/img/zk/state_dia.jpg)

上图中有很多*event*。Go的zk库中，创建一个连接的之后，会返回一个event的channel，这个channel在这个conn被client主动`Close`之前是总是有效的，它用于接收上图中的各种event。

# Session Expiration

会话超时会导致：

1. client的连接channel会收到*expiration event*，并且会话状态迁移到**CLOSE**
1. client创建的eznode会断开
1. client设置的watcher失效

会话超时的定义是：

> When a client (session) becomes partitioned from the ZK serving cluster it will begin searching the list of servers that were specified during session creation. Eventually, when connectivity between the client and at least one of the servers is re-established, the session will either again transition to the "connected" state (if reconnected within the session timeout value) or it will transition to the "expired" state (if reconnected after the session timeout). 

我们可以得出以下结论：

1. 会话超时的判定是由zk server端判定的
2. 如果zk集群宕机，一段时间后重启：zk server不会认为client的session超时
3. 如果是client宕机，一段时间后重启：仅当宕机时间小于**timeout**，才不算超时
4. 如果是client和zk集群网络分区：同3

另外：

> It is not advisable to create a new session object (a new ZooKeeper.class or zookeeper handle in the c binding) for disconnection. The ZK client library will handle reconnect for you. ... Only create a new session when you are notified of session expiration (mandatory).

也就是说，仅当收到*expiration event*之后，才需要重新创建新的会话，这里所谓的重新创建新的会话不是指重连（因为client库都在**timeout**后就会不断尝试重连）。而是指你应该重新"恢复"会话，例如创建eznode，设置watcher等。

timeout的值在`zk.Connect()`中作为参数传入，但是需要注意的是：

>  The client sends a requested timeout, the server responds with the timeout that it can give the client. The current implementation requires that the timeout be a minimum of 2 times the tickTime (as set in the server configuration) and a maximum of 20 times the tickTime. The ZooKeeper client API allows access to the negotiated timeout.

也就是说最终的timeout值不一定是设置的那个值

# 实验

见[这里](https://github.com/magodo/go_snippet/tree/master/zookeeper)

# 分布式的应用

## 抢占znode实现高可用


