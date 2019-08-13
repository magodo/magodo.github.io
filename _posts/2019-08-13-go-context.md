---
layout: "post"
title: "浅谈 Go context"
categories: "blog"
tags: ['golang']
published: True
comments: true
script: [post.js]
excerpted: |
    浅谈 Go context 
---

* TOC
{:toc}

> Ironic that people think context.Context is only for distributed system engineers given the fact context package is only for in-process propagation.
>
> - J.B.D

[详见](https://medium.com/@rakyll/context-propagation-over-http-in-go-d4540996e9b0)

众所周知，context 用于统一管理某个请求，不论是cancel, timeout，或者value的存取。这些功能在单个进程内部的goroutine之间是完全支持的。

但是，一旦跨进程传播，这些功能都需要应用在发送和接收部分的代码自己处理。例如，使用http协议进行通讯，那么发送和接收方可以将timeout, value保存在http header中。

所以，在分布式系统中使用context的时候，开发者需要清除当前使用的框架对context的处理的实现。以下举几个例子：

## go-micro

在当前的go-micro(v1.8.3)中，默认的实现：rcp server/client ，

### timeout

client在发送请求的时候会在请求体中将当前context中的timeout加入到请求体，在server端会读取该值用于构造server端的context. 


### metadata (the only key in value)

此外，go-micro在传递的请求体中可以加入预定义的metadata，这些metadata存在一个map里，而这个map以一个特殊的key存在context的value中，在client端会加入到请求体，server端会恢复到context中.

### cancel

没有实现cancel的传播。即client端进行cancel不影响server端继续执行。

### go-micro summary

这种模式下，我们只要保证请求的发起方带上超时时间即可，后续的服务都不需要对每个接口定义超时。另外，对于异步的请求，当server返回之后，client端执行cancel也不会影响server当前正在处理的异步操作。

## grpc

对grpc不是太熟悉，但是听说grpc在似乎支持客户端cancel的传播 (TODO: verify).
