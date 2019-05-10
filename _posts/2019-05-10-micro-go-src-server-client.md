---
layout: "post"
title: "go-micro 源码解析 - server & client"
categories: "blog"
tags: ['golang']
published: true
comments: true
script: [post.js]
excerpted: |
    不知道能不能看得懂，哈哈！Let's see！
---

* TOC
{:toc}

所有接触过`go-micro`的同学一定都看过官方的那个greeter的示例吧，这篇文章尝试着探索一下greeter的server和client背后到底做了哪些事情。

# Greeter example

greeter代码如下所示：

**server**

```go
package main

import (
	"log"
	"time"

	hello "github.com/micro/examples/greeter/srv/proto/hello"
	"github.com/micro/go-micro"

	"context"
)

type Say struct{}

func (s *Say) Hello(ctx context.Context, req *hello.Request, rsp *hello.Response) error {
	log.Print("Received Say.Hello request")
	rsp.Msg = "Hello " + req.Name
	return nil
}

func main() {
	service := micro.NewService(
		micro.Name("go.micro.srv.greeter"),
		micro.RegisterTTL(time.Second*30),
		micro.RegisterInterval(time.Second*10),
	)

	// optionally setup command line usage
	service.Init()

	// Register Handlers
	hello.RegisterSayHandler(service.Server(), new(Say))

	// Run server
	if err := service.Run(); err != nil {
		log.Fatal(err)
	}
}
```

**client**

```go
package main

import (
	"context"
	"fmt"

	hello "github.com/micro/examples/greeter/srv/proto/hello"
	"github.com/micro/go-micro"
)

func main() {
	// create a new service
	service := micro.NewService()

	// parse command line flags
	service.Init()

	// Use the generated client stub
	cl := hello.NewSayService("go.micro.srv.greeter", service.Client())

	// Make request
	rsp, err := cl.Hello(context.Background(), &hello.Request{
		Name: "John",
	})
	if err != nil {
		fmt.Println(err)
		return
	}

	fmt.Println(rsp.Msg)
}
```

# Init

从上面的例子中可以看到，在`go-micro`中，初始化一个只有一个handler的server只要三行代码：

```go
service := micro.NewService(...)
service.Init(...)
xxx.RegisterXXXHandler(service.Server(), new(XXX))
```

而初始化一个只和一个server交互的client也只要三行代码：

```go
service := micro.NewService(...)
service.Init(...)
c := xxx.NewXXXService("xxx", service.Client())
```

其中，对于client和server，它们的前两行是一样的。它的作用是**初始化service的Options**，其中包括：

- 各种component(e.g. server, client, broker, registry, .etc)
- 某些component的options(e.g. server name, client selector, .etc)
- convention option
- hook option

接下来，我们先看看这两行初始化service的代码背后在做什么，然后再剖析一下server和client特有的那一行代码的作用。

## micro.NewService()

![NewService](/assets/img/go-micro/src-NewService.svg)

## service.Init()

![service.Init](/assets/img/go-micro/src-ServiceInit.svg)

**Take Away**: options设置的优先级为：`cli arguments > env.var. > hardcoded (NewService()/service.Init())`

## server: xxx.RegisterXXXHandler()

(这个方法由`go-micro`的proto plugin生成的代码提供)

server注册其rpc实现（即**handler**）。如果你用的是默认的rpc server(而不是grpc server)，它会加入到内部rpc router的`serviceMap`里面。

## client: xxx.NewXXXService()

(这个方法由`go-micro`的proto plugin生成的代码提供)

拿greeter 的client来说，它的实现如下：

```go
func NewSayService(serviceName string, c client.Client) SayService {
	if c == nil {
		c = client.NewClient()
	}
	if len(serviceName) == 0 {
		serviceName = "go.micro.srv.greeter"
	}
	return &sayService{
		c:           c,
		serviceName: serviceName,
	}
}
```

一般情况下，应该显式地传入`serviceName`和`c`参数。

传入`serviceName`很好理解，因为server的名字很可能不叫这个默认的名字。

而传入`c`的原因是因为在初始化service的时候，你可能设置了各种options。这些options被应用到当时的service内部的`options.Client`对象上。如果，在此时调用`xxx.NewXXXService()`的时候不使用service中的client，而是new了一个新的client，亦或是传入一个`nil`让函数自己创建一个默认的client，那么这个client内部的options都是默认的options，而不是你在初始化service时使用的那些options。笔者昨天就犯了这么一个低级错误，在`NewXXXService()`中调用了`client.NewClient()`，导致registry的设置(consul)都没生效（而是用的默认的mdns），导致selector无法找到server (server注册在consul上)。结果后来花了大半天debug才发现。这也是促使我阅读源码并且写这篇博客的原因。

# Action

看完了基本的初始化流程，我们看看server和client是如何工作的。

## service.Run()

![service.Run](/assets/img/go-micro/src-ServiceRun.svg)

## cl.XXX()

![client.Run](/assets/img/go-micro/src-ClientRun.svg)
