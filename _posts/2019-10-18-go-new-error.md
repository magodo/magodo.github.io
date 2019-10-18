---
layout: "post"
title: "go1.13 错误处理"
categories: "blog"
tags: ['go-new-error']
published: True
comments: true
script: [post.js]
excerpted: |
---

* TOC
{:toc}

Go1.13对错误处理做了一定的改善，语言层面主要有以下几个改动：

- 定义一个error类型的时候除了需要实现 `Error()` 函数以外，用户也可以 **额外** 实现一个 `Unwrap()` 函数，从而使`errors`库新增的函数（后面提起）可以生效
- `errors`包定义了新的函数：
  - `errors.Is()` 判断输入的error参数是否是某个sentinel error变量。如果该输入的error参数wrap N层error的话，此函数会一路展开，直到找到sentinel error或者展开到底了
  - `errors.As()` 将输入的error参数转或者wrap的参数，一旦符合指定的输入接收error参数的类型（第二个参数），就将其赋值给接收error参数
- `fmt`包的`fmt.Errorf()`定义了新的verb：`%w`，用来wrap error

这里参考[官方博客](https://blog.golang.org/go1.13-errors)，主要想整理一下在新的错误处理惯例下，package 作者如何判断何时应该返回wrapped error，literal error: 

如果底层的错误对于上层用户有意义，并且该底层错误不会由于日后重构而改变。则可以将底层error wrap后返回。

正面例子：

- 一个parser函数，接收用户传入的`io.Reader`。如果parser函数由于`io.Reader`的`Read()`函数报错，那么wrap该错误并返回是合理的。因为， `io.Reader`是用户提供的，所以用户依赖于该错误是合理的

中立例子：

- 一个query函数，（当前）会查询数据库。当查询数据库的时候出错了，可能会返回 `sql.XXXError`，这时候是否wrap该错误取决于是否承诺用户这个API总是使用数据库去query。如果不是的话，例如未来使用RPC去通过别的服务做查询，那么就不应该wrap该错误。而是返回一个自定义的error/literal error(使用 `fmt.Errorf(%v)`)