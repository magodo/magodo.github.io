---
layout: "post"
title: "go-micro 防坑指南"
categories: "blog"
tags: ['golang']
published: true
comments: true
script: [post.js]
excerpted: |
    记录使用go-micro的过程中的一些经验和教训...
---

* TOC
{:toc}

[go-micro](https://micro.mu/)是一个使用go语言实现的微服务生态系统，之所以称为生态系统是因为它既有用于开发微服务的框架(*go-micro*)，也有微服务相关的一整套工具(*micro*)。

这个项目历经四年的开发，在2019年3月终于发布了第一个正式release(1.0.0)，宣称足够稳定。

相比与其他go语言的微服务框架(e.g. gizmo, go kit)而言，这个框架的优势是允许开发人员足够快的开发一个系统，并且提供各种中间件的选择性。但是，它的坏处是它是一个opinionated的框架，给开发者的自定义程度较低。

下面，记录我在使用这个项目进行微服务开发过程中的经验和教训。


# go-micro 实现proto的rpc接口

和直接使用grpc相比的一个区别是，使用`micro/protoc-gen-micro`生成出来的*xxx.micro.go*文件可以让service的实现者去实现service对应的`handler`，然后在server/client的启动过程中调用`RegisterXXXServiceHandler()`来注册你的实现。

特别地，这些`handler`将rpc的接口原型中的response作为输入参数，而返回值仅为一个`error`。

例如：

针对如下的rpc：

```proto
service Example {
    rpc Call(Request) returns (Response) {}
}
```

需要实现的`handler`为：


```go
type ExampleHandler interface {
	Call(context.Context, *Request, *Response) error
}
```


# client/server 使用grpc

默认情况下，go-micro的client和server使用`go-micro`库中的server和client的实现，它们是基于http协议进行通讯。

除了这个默认的实现，它也允许用户使用grpc来创建client和server。使用的方法为：

1. proto文件的编译：在编译proto的时候将`--go_out=<some_dir>`改为`--go_out=plugins=grpc:<some_dir>`
2. server和client的创建：在创建server/client的时候使用`micro/go-grpc`提供的`NewService()`(而不是`micro/go-micro`的`NewService()`)
3. (仅当前版本) 需要在go.mod中加入：

        replace github.com/testcontainers/testcontainer-go => github.com/testcontainers/testcontainers-go v0.0.0-20181115231424-8e868ca12c0f
        replace github.com/golang/lint => github.com/golang/lint v0.0.0-20190227174305-8f45f776aaf1

当然，client和server要配合使用，否则由于协议不同而无法通讯。

# selector

详见[官方blog](https://micro.mu/blog/2016/05/15/resiliency.html)

go-micro的selector是基于服务发现做负载均衡和流量控制的，它主要有以下的功能：

- filter: 通过预设的filter（包括：label-based, endpoint-based, version-based）或者自定义的filter进行服务的过滤。例如：有一个服务名为micro.service.foo，用户可以部署不同版本的该服务。然后，在客户端调用的时候根据version进行filter，获取想要调用的版本
- strategy: 对于filter的结果，根据不同算法进行负载均衡

另外，它内置一些辅助功能，帮助用户更好地实现一个健壮的微服务系统，包括：

- selection cache: 避免了每次服务发现都要查询后端，同时，通过一个watcher做必要的reload
- blacklisting node：自动将问题节点列入黑名单，过一段时间再将其解除黑名单
- timeout & retries
- connection pool

以下，主要根据官方的例子来展示一下怎么使用filter。参考 *micro/examples/server/wrapper* 和 *micro/examples/client/dc_filter*，其中server端在metadata中加入了`datacenter=local`:

```go
func main() {
	// optionally setup command line usage
	cmd.Init()

	md := server.DefaultOptions().Metadata
	md["datacenter"] = "local"

	server.DefaultServer = server.NewServer(
		server.WrapHandler(logWrapper),
		server.WrapSubscriber(logSubWrapper),
		server.Metadata(md),
	)
    ...
```

client端则在调用的时候进行metadata的过滤：

```go
    ...
    // create context with metadata
    ctx := metadata.NewContext(context.Background(), map[string]string{
        "datacenter": "local",
    })

    ...

    // Call service with the context
    if err := client.Call(ctx, req, rsp); err != nil {
        fmt.Println("call err: ", err, rsp)
        return
    }
    ...

// wrap the client, do filter before actual call
func (dc *dcWrapper) Call(ctx context.Context, req client.Request, rsp interface{}, opts ...client.CallOption) error {
    // fetch metadata from context
	md, _ := metadata.FromContext(ctx)

    // define a filter closure to only select server with matching metadata configured
	filter := func(services []*registry.Service) []*registry.Service {
		for _, service := range services {
			var nodes []*registry.Node
			for _, node := range service.Nodes {
				if node.Metadata["datacenter"] == md["datacenter"] {
					nodes = append(nodes, node)
				}
			}
			service.Nodes = nodes
		}
		return services
	}

    // extend options with SelectionOption
	callOptions := append(opts, client.WithSelectOption(
		selector.WithFilter(filter),
	))

    // do actual call (with extended option passed in)
	return dc.Client.Call(ctx, req, rsp, callOptions...)
}
```

当然，客户端的filter可以直接调用默认提供的`FilterLabel()`，而不是像例子中的那样自己实现一套，如下：

```go
// wrap the client, do filter before actual call
func (dc *dcWrapper) Call(ctx context.Context, req client.Request, rsp interface{}, opts ...client.CallOption) error {
    // fetch metadata from context
	md, _ := metadata.FromContext(ctx)

    // extend options with SelectionOption
	callOptions := append(opts, client.WithSelectOption(
		selector.WithFilter(selector.FilterLabel("datacenter"), md["datacenter"]),
	))

    // do actual call (with extended option passed in)
	return dc.Client.Call(ctx, req, rsp, callOptions...)
}

```

有了filter，我们可以做服务的金丝雀发布。

而金丝雀发布又有很多做法。

一种做法是完全布一套新的服务，然后从最上层进行流量控制。这种做法比较简单可靠。

另一种做法是通过某种机制对每一个服务流量控制。我的一个设想大致如下：

![canary](/assets/img/go-micro/canary.png)

假设我们对某个feature按用户进行灰度，我们必须知道这个feature是由哪几个服务提供的，并且这个feature要应用于哪些用户。将这些记录到consul。

每个服务都内置一段作为client的filter逻辑，这套逻辑包括三部分：

- 启动时从consul载入这些feature的kv信息；
- 启动一个watcher来watch consul中的feature相关的kv的更新
- 每当要发起一个请求，根据这个请求的服务接收方(`service`)和这个请求对应的`user_id`，进行对服务的基于label的filter

# wrapper

go-micro 中的wrapper一般是在`micro.NewService()`的时候传入的，大致有以下几种：

- **ClientWrapper**: 调用`micro.WrapClient()`返回，该函数的输入参数可以是多个wrapper，这些wrapper的作用是在封装当前的client，所以，它们会返回一个新的client。需要注意的是，这个封装的顺序和传入的顺序**相反**（实际上这样更好理解，一个请求发生的时候，先调用第一个传入的wrapper内的逻辑，再调用第二个，以此类推，最后调用实际的client逻辑）:

    ```go
    func WrapClient(w ...client.Wrapper) Option {
        return func(o *Options) {
            // apply in reverse
            for i := len(w); i > 0; i-- {
                o.Client = w[i-1](o.Client)
            }
        }
    }
    ```

    例如，你想对一个client加入以下两个wrapper：
    
    - tracing wrapper: 封装了client的`Call()`等函数，这些函数会在调用前创建一个span，并且将span作为context传入rpc的调用
    - log wrapper: 封装了client的`Call()`等函数，这些函数会在调用前从ctx中获取span，并且使用span记录log

    显然，这里的*log wrapper*依赖于*tracing wrapper*，也就是说，实际client的`Call()`应按先完成`tracing wrapper`的工作，然后再完成`log wrapper`的工作。所以传入的顺序如下:

    ```go
    service := micro.NewService(
        micro.WrapClient(
            opentracingWrapper.NewClientWrapper(tracer),
            logWrap,
        ),
    )
    ```

- **CallWrapper**: ClientWrapper的简化版，仅影响`Client.Call()`

- **HandlerWrapper**: 调用`micro.WrapHandler()`返回，该函数的输入参数可以是多个wrapper，这些wrapper的作用是*返回一个返回封装当前server的handler函数的函数*，并且将这些函数记录在server的options中。在之后实际接收到某个请求的时候，会按照options中记录的wrapper的逆序进行应用:

    ```go
    // FROM: go-micro/options.go
    // WrapHandler adds a handler Wrapper to a list of options passed into the server
    func WrapHandler(w ...server.HandlerWrapper) Option {
        return func(o *Options) {
            var wrappers []server.Option

            for _, wrap := range w {
                wrappers = append(wrappers, server.WrapHandler(wrap))
            }

            // Init once
            o.Server.Init(wrappers...)
        }
    }

    // FROM: go-micro/server/rpc_server.go
    func (s *rpcServer) ServeConn(sock transport.Socket) {
        ...

        // create a wrapped function
        handler := func(ctx context.Context, req Request, rsp interface{}) error {
            return r.ServeRequest(ctx, req, rsp.(Response))
        }

        for i := len(s.opts.HdlrWrappers); i > 0; i-- {
            handler = s.opts.HdlrWrappers[i-1](handler)
        }

        ...
    }
    ```

    所以，和`ClientWrapper`一样，传入的顺序即反映了请求被执行的顺序。


- **SubscriberWrapper**：broker subscription 的wrapper，如上。


# option

go-micro 中的几乎每一个组件的包（例如：`client`, `server`, `broker`, `registry`...）都会提供一个*options.go*的文件，其中包含了这个包在初始的时候所支持的选项。

例如，默认的`server`在收到关闭的信号(SIGTERM, SIGKILL, SIGINT)的时候只会停止监听，但是不会等待当前正在进行的那些请求结束。如果，你希望你的服务支持graceful shutdown的话，就要在创建`micro.NewService()`的时候带上`server`包提供的`Wait(bool)`选项。

# broker

对于server，起一个subscriber的流程如下：

1. `micro.NewService()`
2. `service.Init()`
3. `service.RegisterSubscriber(topic, service.Server(), func/obj)`: 注册subscriber，但是**还没**有连接到*broker
4. `service.Run()`：在`server.Start()`中会去连接broker，至此才算真正的注册了subscriber

(当然，你可以直接用broker，那就需要手动的connect)

对于client，发送一个topic的流程如下：

1. `micro.NewService()`
2. `service.Init()`
3. `micro.NewPublisher(topic, service.Client())`
4. `publisher.Publish(ctx, event)`

(当然，你可以直接用broker，那就需要手动的connect)

有几个点需要注意：

1. 所有的message默认只支持protobuf的二进制格式
2. publisher发送之后如果进程立刻退出，会导致发送实际没有成功。这应该是和broker的库的实现有关（例如：nats的publish是通过`bufio`，然后隔一段时间做flush）
