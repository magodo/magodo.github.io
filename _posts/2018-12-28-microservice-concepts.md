---
layout: "post"
title: "Microservice Concepts"
categories: "blog"
tags: ['micorservice']
published: false
comments: true
script: [post.js]
excerpted: |
    微服务相关的知识笔记...
---

* TOC
{:toc}

# 1. 什么是微服务

微服务的概念实际上非常宽泛，它的目的是通过很多足够小的单元来对外提供完整的服务。

它的好处包括：

1. Technology Heterogeneity: 微服务之间技术栈可以独立
2. Resilience: 微服务拥有一定的容错性，即某个服务挂了不会影响对外暴露的服务
3. Scaling: 可扩展
4. Ease of Deployment: 容易部署
5. Composability: 可复用
6. Optimizing for Replacebility: 单个服务方便替换
7. Organizational Alignment: 适合公司的团队人员分配

一个微服务需要具备以下几个特点：

1. Single Responsibility: 足够小，并且只做一件事情
2. Atonomous：自治。有明确的对外接口，可以单独部署，...

# 2. 微服务的集成

应该要选用怎么样的方法把多个微服务组织起来呢？

## 2.1 微服务间如何交互

### 2.1.1 共享数据库

服务提供外部用户一个DB的访问权，外部用户可以直接访问DB（读写）。这种做法有以下几个弊端：

1. 数据库中的数据格式是服务内部实现的细节，不应该暴露给外部。如果未来有任何对数据库表结构的修改，将会影响到所有使用该表的用户

2. 限定了外部用户使用的技术栈，他们必须使用与该DB对应的driver，如果未来选择使用别的类型的DB（e.g. nosql），会需要用户做修改

3. 内部服务和外部用户可能会修改同一个数据，导致冲突

很明显，这种做法应该被舍弃。

### 2.1.2 提供API

一个微服务应该对外提供一套定义清晰的接口供用户使用。那么，这套接口应该是什么形式的呢？

推荐使用的是：异步（相对于同步），REST（相对于RPC）的API。

在此基础上，提供一套event-based(相对于req/resp)的机制，可以大大解偶其与外部服务.

## 2.2 微服务内部的实现

每个微服务都应该是一个状态机。假设，这个微服务控制个DB实例的生命周期。那么，所有对该服务的请求都应该顺序的执行，并且状态间的迁移有明确的定义。这样，可以防止状态的冲突。

有时候，一个操作需要多个请求才能完成，建议使用reactive extension.
