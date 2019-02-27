---
layout: "post"
title: "Golang sql库"
categories: "blog"
tags: ['golang']
published: true
comments: true
script: [post.js]
excerpted: |
    关于golang sql库的一些知识点...
---

* TOC
{:toc}

以下讨论基于:

    $ go version
    go version go1.10.3 linux/amd64

# 1. sql库和driver的关系

sql库定义了一系列SQL接口，由driver库去实现，达到屏蔽不同DB类型操作细节的目的。

sql库的主要逻辑定义在`sql.go`，接口定义在`driver.go`。

sql库提供一个`Register()`函数，每一个driver库在`init()`的时候需要调用该函数注册自己实现的`Driver`接口，这个接口仅有一个函数：`Open()`

# 2. Open()

sql库中的`Open()`和driver的`Open()`不太一样。

sql库中的`Open()`更像是Init，它初始化了一个对某个DB实例的handle，之后只要对这个handle进行操作即可，但它并没有真正的去尝试连接。

driver中的`Open()`更像是Connect，它是实实在在地往某个DB实例创建一个session。

## 2.1 sql's Open()

`sql.go`中定义了一个`Open()`函数，这个函数会判断driver是否实现了`DriverContext`接口，该接口仅有一个函数：`OpenConnector()`。

如果driver实现了`DriverContext`，那么就会使用它创建一个`Connector`；否则，使用sql库自身定义的`dsnConnector`类型来构造一个`Connector`.

区别在于，实现了`DriverContext`的driver可以自己定义`Connector`的行为. 默认的`dsnConnector`在调用`Connect()`的时候仅仅执行driver的`Open()`函数。

（mysql和pg用的都是默认的`dsnConnector`）

接下来，有了`Connector`后，我们并没有直接使用它去连接DB，而是把用它来构造一个`DB`的结构体:

    func OpenDB(c driver.Connector) *DB {
        ctx, cancel := context.WithCancel(context.Background())
        db := &DB{
            connector:    c,
            openerCh:     make(chan struct{}, connectionRequestQueueSize),
            resetterCh:   make(chan *driverConn, 50),
            lastPut:      make(map[*driverConn]string),
            connRequests: make(map[uint64]chan connRequest),
            stop:         cancel,
        }

        go db.connectionOpener(ctx)
        go db.connectionResetter(ctx)

        return db
    }

接着，会起两个goroutine:

1. 用来接收连接的请求，每当需要Query, Exec,...的时候，这个goroutine负责调用`Connector`的`Connect()`函数（默认就是调用driver的`Open()`）来创建一个session. 

    需要注意的是，并不是所有连接请求都会经过这个goroutine，只有新创建的连接才会经过它；如果是复用已有的连接，则不会经过这个goroutine.

2. 用来接收reset的请求

## 2.2 driver's Open()

现在，看看当一个连接请求来临的时候，发生了什么。

首先，需要回答什么时候会有连接请求。如上所述，sql库的`Open()`并没有向DB实例创建一个连接，而只有当通过`*sql.DB`执行诸如`Query()`，`Exec()`等动作的时候才“可能”去创建连接。

以`Query()`为例，它首先会创建或者复用一个连接，假设现在我们是创建一个新的连接，乐观的情况下，它其实只是调用了driver的`Open()`函数，然后将返回的`Conn`和`*sql.DB`一起，构造一个`sql.driverConn`结构体。

然后，使用这个`*sql.driverConn`进行后续的Query操作。

这个`Open()`函数，大多数driver都会尝试连接DB实例，然后完成一些握手协议之类的。需要注意的是，这里的所有操作都不带context。这意味着，如果你在执行`dial()`的时候阻塞了，那么你将一直阻塞。为了防止这种情况的发生，driver一般都会检查dsn(连接字串)，如果其中带有超时的设置(e.g. mysql: `timeout=5s`, postgres: `connect_timeout=5`)，那么会调用`net.DialTimeout()`去创建连接。

# 3. Query()

2.2节说了`Query()`的前半部分操作，即创建一个到DB实例的连接。接下去，sql库会判断driver的这个`Conn`是否实现了`driverQueryerContext`的接口（实现`QueryContext()`函数），如果有的话，会调用带context的接口；否则，context不会在Query过程中起作用。

# 4. Ping()

`Ping()`和`Query()`的流程类似，会先创建一个连接（调用driver的`Open()`）。当且仅当driver实现了`Ping()`，才会继续调用driver的`Ping()`；否则， 直接退出。

并不是所有的driver都实现`Ping()`，例如，mysql实现了，但是pg没有。

在sql库中对`Ping()`的描述如下：

> Ping verifies a connection to the database is still alive, establishing a connection if necessary.

而实际上(截止: Feb.20.2019)，`pq`没有实现这个接口；而`mysql`的master分支已经比较正确的实现了(最新的tag:v1.4.1中依然存在bug，见[这里](https://github.com/go-sql-driver/mysql/pull/875))，即如果有一个`sql.DB`对象，循环地执行`Ping()`操作，当DB关闭之后，`Ping()`返回失败；而DB再次启动之后，`Ping()`又返回成功，并且这个对象也可以继续使用。

对于`pq`而言，只能通过执行一些命令(e.g. `db.Exec(";")`)来检查DB的连通性了。

**注意**: 这个Ping是指应用层的Ping，例如对于mysql而言，它是先完成了Connection Phase，然后在Command Phase进行的。成功意味着：

- DB在传输层可以连接
- 帐号名密码匹配
- DB连接没有达到上限
- ...

如果你只是想判断DB服务是否启动，也即DB是否监听响应的IP和端口，而不在乎DB是否可以对外提供服务，那么你应该通过以下几种方式中的一种：

- 使用`net.Dial`。注意，如果你获得的是一个dsn (e.g. 对于mysql: `root:123@tcp(localhost:3306)/foobar`)，那么通过它想获得`net.TCPAddr`需要使用响应driver中的解析函数
- 执行一条dummy命令，仅关注错误类型为`net.OpError`。例如，mysql中执行`select 1`; pg中执行`;`。

# 5. 小结

在和DB实例进行交互的时候，有以下两种情况可能导致阻塞：

1. 第一次连接DB实例
2. 在成功连接实例以后执行操作（ping, query, exec,...）

为了防止第一种，在dsn(connstring)中加入超时；为了防止第二种，在操作时加入context.

