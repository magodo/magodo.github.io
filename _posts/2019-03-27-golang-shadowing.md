---
layout: "post"
title: "golang 之 变量隐藏(shadowing)"
categories: "blog"
tags: ['golang']
published: True
comments: true
script: [post.js]
excerpted: |
    谈一谈什么是shadowing，以及如何防止...
---

* TOC
{:toc}

# shadowing

golang里面的`:=`符号给我们带来便利的同时也可能会导致一些意想不到的结果，shadowing就是其中一个。

众所周知，`:=`符号的左边如果是多个变量，那么只要**大于等于1**个变量是为定义的即可使用`:=`来声明一个新变量，并且对那些已经定义的变量仅进行赋值操作。但是，一旦扯上作用域之后，就可能导致shadowing。

举个例子：

{%highlight go %}
package main

import "errors"

func main() {
	bar()
}
func foo() (string, error) { return "", errors.New("foobar") }
func bar() (err error) {
	defer func() {
		if err != nil {
			print("error occurs")
		}
	}()

    s, err := foo()
    _ = s   // just for compiling
    _ = err // just for compiling

	return
}
{%endhighlight%}

输出：

```
error occurs
```

但是，如果稍微做一点修改，将`foo()`的调用放到作用域中：


{%highlight go %}
package main

import "errors"

func main() {
	bar()
}
func foo() (string, error) { return "", errors.New("foobar") }
func bar() (err error) {
	defer func() {
		if err != nil {
			print("error occurs")
		}
	}()

    {
        s, err := foo()
        _ = s   // just for compiling
        _ = err // just for compiling
    }

	return
}
{%endhighlight%}

这时不会有任何输出，那是因为`foo()`表达式左边的`err`与defer函数内的`err`不是同一个变量。

那么，如何检测这种潜在的bug呢 🤔 .....................................   [`shadow`](https://godoc.org/golang.org/x/tools/go/analysis/passes/shadow/cmd/shadow)!

# shadow detects shadowing

`shadow` 是一个go的命令，需要安装：

```
go install golang.org/x/tools/go/analysis/passes/shadow/cmd/shadow
```

接下来就可以使用`go vet -vettool=$(which shadow)`来调用了。

但是，`shadow`默认仅会检查函数返回值中的变量是否被shadow，而不会检测处上面的那个bug。幸好，它提供了一个`-strict`选项。因此，对于我们的这个问题可以这么去检查：

```
💤  test  go vet -vettool=$(which shadow) -strict
# _/tmp/test
./main.go:16:6: declaration of "err" shadows declaration at line 9
```

不过，有时候你确实想在新的作用域中创建一个与上一层的某个变量同名的变量。但是，此时`shadow -strict`依然会报错。这种报错我们可以认为是误报，你或许可以忽略。然而，设想一下你有很多源文件，你很自信这些文件里面现存的报错都是误报。但是，继续开发一段时间以后，再跑一遍`shadow`，你怎么样从这时输出的报错中筛选出那些这段时间开发引入的真正的shadowing bug呢🤔 

似乎没有什么办法。。。

换个思路，我们应该从一开始就消灭所有的报错，即使它是误报。我们也可以使用重命名变量的方式来解决。但是，面对已有的海量代码，重命名很麻烦，这时候[`godoctor`](http://www.gorefactor.org/doc.html#godoctor-vim) is your friend 👻
