---
layout: "post"
title: "golang reflect"
categories: "blog"
tags: ['golang']
published: True
comments: true
script: [post.js]
excerpted: |
    感觉每次要用到reflect都要看一遍官网，是时候用自己的语言组织一下对reflect的理解...
---

* TOC
{:toc}

# interface representation

golang中的interface类型的变量在内部实现表现为**(value, type)**对，其中：

- **value**: 对应实际实现这个interface的对象
- **type**：实现interface的对象的实际类型

例如：

```go
var r io.Reader
tty, err := os.OpenFile("/dev/tty", os.O_RDWR, 0)
if err != nil {
    return nil, err
}
r = tty
```

那么，`r`对应的**(value, type)** pair 是(`tty`,`*os.File`)。

需要注意的是，interface实现层面的这两个信息代表的是它的**实际**信息，在语法层面，`r`的类型总是静态的(static type)。所以，如果想把`r`赋值给类型为`io.Writer`的变量是需要做type assertion才行。引用[官网](https://blog.golang.org/laws-of-reflection)的一段话：

> Some people say that Go's interfaces are dynamically typed, but that is misleading. They are statically typed: a variable of interface type always has the same static type, and even though at run time the value stored in the interface variable may change type, that value will always satisfy the interface. 

不过，接下来对于`reflect`的讨论中，interface类型的变量的信息都是实际类型/值。

# three reflect laws

## law 1: Reflection goes from interface value to reflection object

reflect提供2个最基本的类型：`reflect.Value`和`reflect.Type`。`reflect.Value`由`reflect.ValueOf()`函数返回，`reflect.Type`既可以由`reflect.TypeOf()`返回，也可以由`reflect.Value`的`.Type`属性获得，所以我们一般只用`reflect.ValueOf()`就够了。

这里的`Value`和`Type`其实获取的正是上一节所说的interface实现中的实际值和类型 (因为所有调用`reflect.ValueOf()`的输入参数都被先赋值给空的interface，接下来返回这个interface变量的实际值：`func ValueOf(i interface{}) Value` )。所以，不论是某个实际类型的变量，还是某个interface类型的变量，`reflect`都能获得它们实际的类型和值。

另外，`Value`和`Type`都有一个`Kind()`方法，它返回的实际上是当前实际类型的**最左边的底层类型**。举例来说：

```go
type Foo []int

f := Foo{}
```

这里`f`的`Kind`是`slice`。其中，**底层类型**为`[]int`，即`slice of int`。所以**最左边的底层类型**即为`slice`.

总结一下，`Type`和`Kind`的关系大致如下：

    ? (static type, including interface{}) -- reflect.Type() --> concrete type (of interface{}) -- reflect.Kind() --> left most underlying type

下面再列几个例子：

```go
package main

import (
	"fmt"
	"reflect"
)

type CustomType struct{}
type CustomType2 []struct{}
type AliasType CustomType

func main() {
	foo := []interface{}{
		main,
		100,
		[]int{1},
		CustomType{},
		CustomType{},
		AliasType{},
	}
	for _, i := range foo {
		v := reflect.ValueOf(i)
		fmt.Println(v.Type(), v.Kind())
	}
}
```

输出：

```
func() func
int int
[]int slice
main.CustomType struct
main.CustomType struct
main.AliasType struct
```

`reflect`的Value还提供了一组`getter/setter`，它们设置或返回的总是该类型能代表的最大值。例如：`v.Uint()`返回的static type为`uint64`.

## law 2: Reflection goes from reflection object to interface value.

这是law1的反向，即已有一个`reflect.Value`，可以调用它的`Interface()`方法返回`interface{}`类型的变量。`Interface()`将这个`Value`的`Value`和`Type`组装成**(value, type)** pair的inerface返回。

然后，你就可以通过type assertion 把它转变成对应的实际类型 （或者其他它实现了的interface类型）。

## law3: To modify a reflection object, the value must be settable.

`settable`是`Value`的一个属性，只有待set的对象是一个指针，你才可以set它的`Elem()`。

举例如下：

```go
a := "abc"

vp := reflect.ValueOf(&a)
v := vp.Elem()
println(v.CanSet())
```

输出：`true`

其实这整个过程和传递一个指针到某个函数，并且被该函数修改其指向的对象是一样的：

- 传递指针到函数，等价与传递指针给`ValueOf()`
- 函数内修改指针指向的内容的时候要解引用，等价与调用`Value.Elem()`

特殊地，对于一个结构体的指针的`Value.Elem()`，它的成员是settable或者`Interface()`方法的调用前提是，它们必须是是exported（即大写）的。如果不是exported，那么虽然你可以通过`Value.Field()`来引用它们，但是任何对该field的操作都会panic。

# make function

`reflect`可以用来做decorator，以下例子是把一个同步的函数修饰成异步的函数：

```go
package main

import (
	"errors"
	"log"
	"reflect"
)

func MakeAsyncFunction(ch chan error, f interface{}) interface{} {
	vf := reflect.ValueOf(f)
	wrapperF := reflect.MakeFunc(
		vf.Type(),
		func(in []reflect.Value) []reflect.Value {
			go func() {
				out := vf.Call(in)
				var err error
				err, _ = out[0].Interface().(error) // nil interface can't do type assertion
				ch <- err
			}()

			// tricky way to construct `Value` with (value, type) of (nil, error)
			// see: https://stackoverflow.com/questions/51092352/how-can-i-instantiate-a-nil-error-using-golangs-reflect
			return []reflect.Value{reflect.Zero(reflect.TypeOf((*error)(nil)).Elem())}
		},
	)
	return wrapperF.Interface()
}

func main() {
	c := make(chan error, 1)

	f := MakeAsyncFunction(c, foo).(func(int) error)
	err := f(1)
	if err != nil {
		log.Fatal(err)
	}
	err = <-c
	if err != nil {
		log.Fatal(err)
	}

	b := MakeAsyncFunction(c, bar).(func(int, int) error)
	err = b(1, 2)
	if err != nil {
		log.Fatal(err)
	}
	err = <-c
	if err != nil {
		log.Fatal(err)
	}
}

func foo(i int) error {
	log.Printf("foo: %d\n", i)
	return nil

}

func bar(i int, j int) error {
	log.Printf("bar: %d, %d\n", i, j)
	return errors.New("bar error")

}
```
