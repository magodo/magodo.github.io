---
layout: "post"
title: "Dart 之 异步"
categories: "blog"
tags: []
published: True
comments: true
script: [post.js]
excerpted: |
    聊聊Dart的异步操作...
---

* TOC
{:toc}

# Intro

默认情况下Dart程序只有**一个**控制流([isolate](https://api.dartlang.org/stable/2.2.0/dart-isolate/dart-isolate-library.htm://api.dartlang.org/stable/2.2.0/dart-isolate/dart-isolate-library.html))，如果有什么耗时的操作被发起，那么整个程序会被阻塞。

异步操作允许你的程序仅仅是发起一个操作，无需阻塞地等待该操作的结果，而是可以接下去做别的事情。

Dart语言中使用`future`来表示一个异步操作。一个`future`在程序中的表现形式是一个`Future<T>`的对象（其中`T`代表这个异步操作最终的结果的类型，如果不返回结果，则`T`为`void`）。

当一个`future`被发起，它会：

- 将对应的函数（和定义的回调函数）放入**Event Loop**，然后返回一个未完成的`Future<T>`对象
- 当这个`future`后续从消息队列被取出并处理完毕后，会返回一个结果或者错误

要了解异步的本质，我们需要先了解Dart的Event Loop.

# Event Loop

Dart程序的运行顺序如下图所示：

![event loop](/assets/img/dart_async/event-loop-and-main.png)

先运行`main()`，结束后，程序并不是立马退出，而是挨个处理消息队列中的事件。

## Event queue vs Microtask queue

Dart程序中有两类队列：

- **event queue**: （低优先级）用于接收各种I/O，鼠标，绘画，定时任务，消息等系统事件，也用于接收程序中发起的事件（通过`future`）
- **microtask queue**：（高优先级）用于接收那些希望在控制流返回给**event queue**之前完成的异步事件，当前仅可以接收程序中发起的事件

因此，我们有以下更细化的程序运行的活动图：

![both queues](/assets/img/dart_async/both-queues.png)

**NOTE**: 我们虽然可以预测不同的消息执行的顺序，但是不能预测它们真正被执行的时间点。因此，在Dart中如果发起了一个延时N秒的异步操作，它并不能保证N秒以后一定被执行（例如，这个异步操作之前有另一个异步操作需要更长的时间）。

## How to schedule a task

如果是发起一个*event queue*的任务，使用`future`的操作原语，包括：`Future`API和`await`（下面会细说）。

如果是发起一个*microtask queue*的任务，使用`scheduleMicrotask()`。但是，由于[9001](https://github.com/dart-lang/sdk/issues/9001)和[9002](https://github.com/dart-lang/sdk/issues/9002)这两个bug，导致：

> the first call to scheduleMicrotask() schedules a task on the event queue; this task creates the microtask queue and enqueues the function specified to scheduleMicrotask(). As long as the microtask queue has at least one entry, subsequent calls to scheduleMicrotask() correctly add to the microtask queue. Once the microtask queue is empty, it must be created again the next time scheduleMicrotask() is called.

另外，有以下几个要注意的点：

1. The function that you pass into Future’s `then()` method executes immediately when the Future completes. (The function isn’t enqueued, it’s just called.)
1. If a Future is already complete before `then()` is invoked on it, then a task is added to the microtask queue, and that task executes the function passed into then().
1. The `Future()` and `Future.delayed()` constructors don’t complete immediately; they add an item to the event queue.
1. The `Future.value()` constructor completes in a microtask, similar to #2.
1. The `Future.sync()` constructor executes its function argument immediately and (unless that function returns a Future) completes in a microtask, similar to #2.

# Async

Dart提供了两套方法让你写基于`future`的异步代码，分别是`async`和`await`。

举一个例子：

有两套不相关的操作: 

- `fetchFoo`，然后`processFoo`
- `doBar`

理论上它们是可以并发执行的，如果没有异步，我们只能线性地执行它们，代码如下：

```dart
String fetchFoo() {
    print("Fetching foo..."); // time consuming
    return "foo";
}
void processFoo(String foo) {
    print("Process $foo");
}

void doFoo() {
    processFoo(fetchFoo);
}

void doBar() {
    print("Doing bar...");
}

void main() {
    print("main starts");
    doFoo();
    doBar();
    print("main ends");
}
```

输出：

```
main starts
Fetching foo...
Process foo
Doing bar...
main ends

```

## Future API

很显然，对于`Foo`的操作，我们可以通过异步的方式执行。改动如下：


```dart
String fetchFoo() {
    print("Fetching foo..."); // time consuming
    return "foo";
}
void processFoo(String foo) {
    print("Process $foo");
}

void doFoo() {
    Future<String>(() => fetchFoo()).then((v) => processFoo(v));
}

void doBar() {
    print("Doing bar...");
}

void main() {
    print("main starts");
    doFoo();
    doBar();
    print("main ends");
}
```

输出：

```
main starts
Doing bar...
main ends
Fetching foo...
Process foo
```

有几个需要注意的点：

1. `doFoo()`的使用使用者(`main()`)没有影响
2. 只需要在原来同步调用的地方使用`Future`来异步调用即可。注意：表达式中的类型（`Future<String>`）是对应其构造函数中传入的函数（`fetchFoo`）的类型所对应的。


## async/await

*下面纯属个人理解，如果有问题请大家给我留言指出，不甚感激😀*

Dart2中引入的`await`语法糖，可以让我们用写同步一样的方式来写异步代码。

首先，我们需要知道`await`的定义是什么：

> In `await expression`, the value of `expression` is usually a Future; if it isn’t, then the value is automatically wrapped in a Future. This Future object indicates a promise to return an object. The value of `await expression` is that returned object. The await expression makes execution pause until that object is available.
> --- [language tour](https://www.dartlang.org/guides/language/language-tour#asynchrony-support)

(吐槽一下，这个定义为什么没有出现在专门讲async的Future那个page里啊🤢)

假设我们有以下的函数：

```dart
void Foo() async {
    await DoA();
    DoB();
}
```

(注意：调用`await`的函数**必须**在一个`async`函数中，这也是`async`的**唯一**作用)

那么沿用上面的定义，其中包含了两种情况：

1. `DoA()`本身返回一个`future`。那么，`Foo()`在执行到`await DoA()`的时候会执行`DoA()`，直到`DoA()`返回`future`。此时，`DoA()`事实上往event loop中加入了一个事件。

    在`DoA()`返回`future`以后，`Foo()`会将`await`表达式后面的语句作为回调append(`then()`)到这个`future`上去（例如这里的`DoB()`）。

2. `DoA()`不返回`future`。那么，接下来会先执行`DoA()`，然后`await`语法糖会将`DoA()`的返回值转变成一个`future`进行返回。

我试图在这里指出的有以下几点：

1. `async`的作用仅为允许被修饰的函数内部调用`await`（[detail](https://github.com/dart-lang/sdk/blob/master/docs/newsletter/20170915.md#synchronous-async-start)），而一个函数到底是同步的还是异步的，完全取决于其内部实现。因此，在不改变API的情况下可以改变一个函数的同步异步属性
2. 上面的例子中的`await DoA()`不是说将`DoA()`整体丢到event loop中等待被执行。而是直接运行`DoA()`直到它返回。从这个角度来看，有没有`await`都一样。但是，`await`的真正作用是它会影响后面的语句；

    - 所有后面的语句会被作为callback append到`await`所触发的那个`future`(事件)。就好比它们都作为那个future的`then`的callback
    - 假设有以下的语句:

        ```dart
        Future<String> foo()  {...}
        final a = await foo();
        print('${a.runtimeTpye}');
        ```

        那么，最终输出的是`String`。虽然`foo()`返回的是个`Future<String>`，但是，当`a`在`await`以后被使用的时候，它就类似`then`中的callback一样，已经被转换成最终的完成时的类型了(`String`)。

因此，我们上面的例子如果改动如下(错误的实现)：

```dart
void doFoo() async {
    await processFoo(await fetchFoo());
}
```

输出：

```
main starts
Fetching foo...
Doing bar...
main ends
Process foo
```

这里在main函数结束前先输出`Fetching foo...`的原因是，这部分代码在`await fetchFoo()`的时候就已经被执行了，只有返回的`foo`才被用来构造`future`并发起异步。而后续的`await processFoo()`则是作为上一个`future`的回调被执行，执行的时候由于使用了`await`，所以会再一次创建一个`future`并发起异步，不过这些动作都是发生在main函数结束以后了，所以`Process foo`显示在`main ends`之后。

完全等价的改动应该是这样：

```dart
void doFoo() async {
    await null;
    await processFoo(fetchFoo());
}
```

(当然，你要在`fetchFoo()`前面加个`await`也没什么关系)

[detail](https://github.com/dart-lang/sdk/blob/master/docs/newsletter/20170915.md#migration)


## Error Handling
