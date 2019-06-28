---
layout: "post"
title: "go-micro: Tracing"
categories: "blog"
tags: ['golang']
published: True
comments: true
script: [post.js]
excerpted: |
    About how to integrate opentracing into go-micro...
---

* TOC
{:toc}

# Intro

Logging is an important factor for any kind of software, it provides visibility into the behavior of running app, helps developer to troubleshoot issue or even profile the app. In context of microservice, aggregating logs among services is hardly enough, you might have to sturcture your log format, add some unique id for each session, and somehow chain all log together whose session id are the same 😶. Seems lots of effort to pay.

Hence comes **tracing**. We will focus on [opentracing](https://opentracing.io/) in this artical:

> Per-process logging and metric monitoring have their place, but neither can reconstruct the elaborate journeys that transactions take as they propagate across a distributed system. Distributed traces are these journeys.

# Integrate with go-micro

But what is the graceful approach to integrate opentracing in [go-micro](https://github.com/micro/go-micro) 🤔

Let's first have a look at what does go micro eco-system provide us?

Essentially, go-micro provides nothing more than the core stuff, it does have a package *utils/log*, but its so simple and has nothing to do with tracing.

On the other hand, go-plugins provides the meat. *wrapper/trace/opentracing* defines four `NewXXXWrapper()` which are used to deliver [span](https://opentracing.io/docs/overview/spans/) across service boundary. User should pass these callback during service initialization, like below: 

```go
// Let's say this is a microservice provideing simple rpc endpoint,
// and migth invoke outgoing rpc call.
srv := micro.NewService(
    micro.Server(server.NewServer(
        server.Name(name),
        server.Version(version),
    )),
    micro.WrapHandler(
        opentracingWrapper.NewHandlerWrapper(tracer),
    ),
    micro.WrapClient(
        opentracingWrapper.NewClientWrapper(tracer),
    ),
)
```

## How to initialize tracer?

The `tracer` used above is initialized before we call `micro.NewService()`. There are kinds of opentracing tracer library out there, and i choose to use [jaeger](https://www.jaegertracing.io/), so just follow the documentation to initialize jaeger trace.

The usual steps are like:

- call `cfg, err := config.FromEnv()` to get the tracer configuration (and customize the cfg as you wish)
- call `cfg.NewTracer()` with options as you wish, to get the `tracer`

## How to utilize the trace in code?

Opentracing provides a comprehensive [demo](https://medium.com/opentracing/take-opentracing-for-a-hotrod-ride-f6e3141f7941). The implementation of this demo give us a good [example](https://github.com/jaegertracing/jaeger/tree/master/examples/hotrod/pkg/log) about how to define a logger interface to work with opentracing.

If you utilize this interface, then during a request session (which should always be bound to a `context.Context`), one can log/trace like below:

```go
logger.For(ctx).Info("hello world")
```

The message will be streamed to both stdout and jaeger agent (or any other tracer infra).

# More Tips

I have some more good practice to show...

## Log every incoming/outgoing request/response

Keep trace of each incoming/outgoing request and response is always useful for debugging and profiling purpose. I write following code to make it out:

```go
// If used with tracing, please ensure this is registerd (by micro.WrapHandler()) after
// micro-plugin's opentracing wrapper: `opentracing.NewHandlerWrapper()`
func LogHandler(fn server.HandlerFunc) server.HandlerFunc {
	return func(ctx context.Context, req server.Request, rsp interface{}) error {
		logBody(ctx, "request(in)", req.Body())
		err := fn(ctx, req, rsp)
		if err != nil {
			For(ctx).Error(
				err.Error(),
			)
			return err
		}
		logBody(ctx, "response(out)", rsp)
		return err
	}
}

// clientLogWrapper implement client.Wrapper
type clientLogWrapper struct {
	client.Client
}

// If used with tracing, please ensure it is registerd (by micro.WrapClient()) after
// micro-plugin's opentracing wrapper: `opentracing.NewClientWrapper()`
func LogClient(c client.Client) client.Client {
	return &clientLogWrapper{c}
}

func (l *clientLogWrapper) Stream(ctx context.Context, req client.Request, opts ...client.CallOption) (client.Stream, error) {
	logBody(ctx, "stream(out)", req.Body())
	return l.Client.Stream(ctx, req, opts...)
}

func (l *clientLogWrapper) Call(ctx context.Context, req client.Request, rsp interface{}, opts ...client.CallOption) error {
	logBody(ctx, "request(out)", req.Body())
	return l.Client.Call(ctx, req, rsp, opts...)
}
func (l *clientLogWrapper) Publish(ctx context.Context, msg client.Message, opts ...client.PublishOption) error {
	logBody(ctx, "publish(out)", msg.Payload())
	return l.Client.Publish(ctx, msg, opts...)
}

func logBody(ctx context.Context, desc string, body interface{}) {
	reqJSON, _ := json.MarshalIndent(body, "", "  ")
	For(ctx).Info(
		desc,
		zap.String("body", string(reqJSON)),
	)
}
```

Then you can use the `LogClient()` and `LogHandler()` functions together with other wrappers during initializing service, like below:


```go
srv := micro.NewService(
    micro.Server(server.NewServer(
        server.Name(name),
        server.Version(version),
    )),
    micro.WrapHandler(
        opentracingWrapper.NewHandlerWrapper(trace.Tracer()),
        LogHandler,
    ),
    micro.WrapClient(
        opentracingWrapper.NewClientWrapper(trace.Tracer()),
        LogClient,
    ),
)
```

Take care of the ordering here, these two wrapper should put after the `opentracingWrapper`s, because the latter wrappers depend on the fact that `span` is available in the `context` around, which is handled by the former wrappers.


## Log every sql statement

I also wrap the `database/sql` to implement sql query and response tracing. The main point is to always use the context variant methods of sql to do the query, and start a span from that context, then use it to do logging stuff. This means we have to abbandom some orm that has no support for context (e.g. gorm, [related issue](https://github.com/jinzhu/gorm/issues/1231)).

The challenge parts of sql tracing are:

- how to parse the sql and replace the placeholder?
- how to trace the response of a query?

These are left as excercise to you 😊

# Conclusion

go-micro has just enough integration with tracing, from a framework's perspective. User like us should make our own effort to meet more customized requirements.

I just give some example about how I make tracing work in my project. If you have any question, hints or tips, please leave a message!
