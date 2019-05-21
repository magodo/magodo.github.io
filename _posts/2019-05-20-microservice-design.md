---
layout: "post"
title: "microservice-设计"
categories: "blog"
tags: ['golang']
published: True
comments: true
script: [post.js]
excerpted: |
    一些关于微服务设计方面的自己的思考...
---

* TOC
{:toc}

# 服务协同模式：Orchestration or Choreography

Orhchestration模式是指，有一个中心的控制模块协调多个服务模块（同步或异步）完成一个用户请求。这要求控制模块需要知道所有它依赖的服务模块的信息(e.g. service name)，有一定程度的耦合。

例如：

![orchestration](/assets/img/microservice/orchestration.png)

Choreography模式则没有这么一个中心的控制模块，一个请求被接收到并广播到系统中，系统中关心该请求的服务模块会进行处理，并且不必要将结果告知某个中心的控制模块。这带来的好处是使模块间完全解偶。

例如：

![choreography](/assets/img/microservice/choreography.png)

Choreography的模式有个问题是下如果某个服务模块没有接收到请求，或者处理该请求的过程中发生异常，由于没有控制模块的存在，默认情况下该异常无法被告警或者补救。因此，需要额外加入一套监控系统监控每一个请求是否被正确处理。

# 异步逻辑

服务之间的异步逻辑有多种实现方式：

- 服务双方约定好协议，A提供一个service endpoint，同时要求B提供一个callback endpoint:

    ![async-rpc2rpc](/assets/img/microservice/async-rpc2rpc.png)

    缺点：调用者还要定义一个callback endpoint，耦合成🐴

- 使用rpc的server-side stream，返回多个respones。例如，第一个response可以作为接收到请求的feedback，第二个response作为实际的响应

    ![async-rpcstream](/assets/img/microservice/async-rpcstream.png)

    优点：一个请求对应一次或多次响应，并且响应的接受者就是请求的发起者(可以保留上下文)

    缺点：

    - rpc的响应message定义不太直观，例如基于protobuf的rpc可能需要用`oneof`来定义不同的response类型。并且client看到server的rpc定义不能一目了然的知道返回response的顺序，需要额外的文档
    - server没有graceful shutdown，server在返回stream之后，通过stream发送respond之前是可以合法关闭的。

- 使用broker实现不同类型的异步逻辑，这种方式非常灵活，可以实现上面提到的逻辑甚至更多。下面举两个例子：

    **无上下文**

    ![async-pubsub-noctx](/assets/img/microservice/async-pubsub-noctx.png)

    **需要保留上下文**

    ![async-pubsub-ctx](/assets/img/microservice/async-pubsub-ctx.png)

    优点：

    - 解偶
    - 灵活多变，例如： 启动时异步subscribe的callback可以用于处理那些不需要上下文的topic。而那些需要上下文的topic则可以通过临时(参考[这里的reply-to](https://nats.io/documentation/writing_applications/publishing/)) 同步subscribe，然后在需要时阻塞recv；或者，临时异步subscribe，并且其callback可以用闭包的方式使用上下文信息。当然，在操作完之后需要unsubscribe（例如，可以使用nats的auto-unsubscribe功能）
    - server端只需要实现异步接口，client要以同步的方式使用还是异步的方式使用是client端的事情
    - server和client总是不需要保持连接
    - 那些不需要上下文的topic可以通过broker queue group的方式使resp的接受者可以不是req的发起者，从而避免点对点的模式下接收方挂了导致消息丢失(at most once broker only)

    缺点：

    - 需要额外维护broker
    - 如果用的broker是at-most-once类型的（例如；nats），那么需要一套监控系统监控，防止msg被丢掉的情况
    - client对需要上下文的topic做异步接收时没法做graceful shutdown。想象一下某个请求的resp msg被发到了broker，但是这时client端（接收者）重启了，那这个唯一的topic也不会被再次注册，导致消息没有人接收而丢失。
    - client对不需要上下文的topic做异步采用全局subscription的callback时，如果别的client想同步调用该请求，那么server应该避免发送

- rpc和event混用，请求是通过rpc发到server端，server端接收到请求之后立即返回，后续的操作结果通过event返回:

    ![async-rpcevt](/assets/img/microservice/async-rpcevt.png)

    优点：

    - 请求总是保证server接收
    - 对比rpc stream的方式，可以通过broker queue group的方式使resp的接受者可以不是req的发起者(如例子所示)，从而避免必须保持连接的限制

    缺点：

    - 接收依然不保证可靠（对于at-most-once的broker）


**最佳实践?**

好吧，其实微服务里没有一个普适的方案，需要各种方案的结合使用，并且结合你的业务需求。

我只能说说我自己的场景，我的业务需求要求我拥有以下几个属性：

- graceful shutdown
- 有些请求需要保留上下文
- 一个请求对应多个响应
- 慢任务请求处理时间很久，c-s可以不保持连接

最终我选择了rpc-rpcstream 和 rpc-event 这两种交互来满足我的不同需求：

![async-best-practice](/assets/img/microservice/async-best-practice.png)

至于那种long request and complext logic after response，则应该从设计上就避免。

需要注意:

1. rpc server side stream在很多框架（例如: go-micro）中是不支持graceful shutdown的，为了实现发布不影响服务的需求，我们需要自己做点额外的工作
2. rpc event 模式需要实现同步接口（如果有同步需求的话），否则发送请求的client可能不是接收请求的client，导致无法实现同步逻辑。另外，如果服务端的响应比较统一，那么可以通过给同步接口进行封装的方式实现异步接口(结合reflect)，比较方便

例如对于go-micro，我们可以设计一个全局的wait group，在每次client/server处理stream的goroutine启动前将`wg.Add(1)`，并且在该goroutine退出时执行`wg.Done()`。同时，在server启动的时候对go-micro的server注册`Before/AfterStop()`hook，在里面调用`wg.Wait()`.

我之所以选择 rpc-rpcstream，而没有选择rpc pubsub (保留上下文)的原因是，后者在go-micro中无法/难以实现（go-micro暴露的broker接口很有限）.
