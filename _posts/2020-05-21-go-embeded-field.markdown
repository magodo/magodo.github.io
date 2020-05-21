---
layout: "post"
title: "Go - Embedded Field"
categories: "blog"
tags: ['go']
published: True
comments: true
script: [post.js]
excerpted: |
    A short description...
---

* TOC
{:toc}

# The Problem

Given following code:

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
)

type Foo struct {
	A string
	Bar
}

type Bar struct {
	B string
}

func main() {
	var foo Foo
	if err := json.Unmarshal([]byte(`{
  "a": "foo",
  "b": "bar"
}`), &foo); err != nil {
		log.Fatal(err)
	}
    spew.Dump(foo)
}
```

The output is as expected:

```bash
(main.Foo) {
 A: (string) (len=3) "foo",
 Bar: (main.Bar) {
  B: (string) (len=3) "bar"
 }
}
```

However, if one day some one implement the `json.Unmarshlaler` for `Bar` as below:

```go
func (v *Bar) UnmarshalJSON(data []byte) error {
	v.B = string(data)
	return nil
}
```

The output of above code will be like:

```bash
(main.Foo) {
 A: (string) "",
 Bar: (main.Bar) {
  B: (string) (len=30) "{\n  \"a\": \"foo\",\n  \"b\": \"bar\"\n}"
 }
}
```

Apparently, this is not the expected result. But why?

One fact is stated in the document of `encoding/json`:

> To unmarshal JSON into a value implementing the Unmarshaler interface, Unmarshal calls that value's UnmarshalJSON method, including when the input is a JSON null.

Another fact is that as `Foo` embeds `Bar`, it means `Foo` "inherit" (actually method promote) the method of `Bar`. To be more precise of what happens here, we need to introduce following two concepts from Go Spec:

1. [**embedded field of structure**](https://golang.org/ref/spec#Struct_types):

    ```
    StructType    = "struct" "{" { FieldDecl ";" } "}" .
    FieldDecl     = (IdentifierList Type | EmbeddedField) [ Tag ] .
    EmbeddedField = [ "*" ] TypeName .
    Tag           = string_lit .
    ```

2. [**method set**](https://golang.org/ref/spec#Method_sets) and **method promote**:

    > A field or method f of an embedded field in a struct x is called **promoted** if x.f is a legal selector that denotes that field or method f.
    >
    > Promoted fields act like ordinary fields of a struct except that they cannot be used as field names in composite literals of the struct.
    > Given a struct type S and a defined type T, promoted methods are included in the method set of the struct as follows:
    >
    > - If S contains an embedded field T, the method sets of S and *S both include promoted methods with receiver T. The method set of *S also includes promoted methods with receiver *T.
    > - If S contains an embedded field *T, the method sets of S and *S both include promoted methods with receiver T or *T.

So in our case, the method of `Bar` is promoted into the method set of `Foo`. When `json` is going to unmarshal for a type T, it will check if T implemented `Unmarshaler`, if yes, then it will just use the `UnmarshalJSON()` found in the method set of T.

That's what happening in our case, the json object is fed into the `Bar`'s `UnmarshalJSON()` and which in turns unmarshal into `Bar` field only.

# A Naive Solution

An intuition idea is to implement `Unmarshaler` for `Foo`, so that when calling `json.Unmarshal`, hoping it will only call the `UnmarshalJSON` of `Foo`.

This idea is the right direction, but the pitfall is that how do we implement it. A naive solution is like below:

```go
func (v *Foo) UnmarshalJSON(data []byte) error {
	return json.Unmarshal(data, v)
}
```

Output:

```
runtime: goroutine stack exceeds 1000000000-byte limit                                                                                                                                   
runtime: sp=0xc020160360 stack=[0xc020160000, 0xc040160000]                                                                                                                              
fatal error: stack overflow  
```

🤯 The stack blows up!

Calm down and re-think a bit, it is not hard to reason about. As we are calling `json.Unmarshal` inside `Foo.UnmarshalJSON()`, passing a `Foo` as parameter. That in turns will call `Foo.UnmarshalJSON()` again, and the loop continues, indefinitely.

# Another Broken Solution

Is there some way to walk around this? A hack is to define another type!

> 💡 **Type Definition vs Type Alias**
>
> - A type definition creates a new, distinct type with the same underlying type and operations as the given type, and binds an identifier to it:
>
>       TypeDef = identifier Type .
>
>   The new type is called a **defined type**. It is different from any other type, including the type it is created from.
>
>   A defined type may have methods associated with it. **It does not inherit any methods bound to the given type**, but the method set of an interface type or of elements of a composite type remains unchanged
>
> - An alias declaration binds an identifier to the given type. Within the scope of the identifier, it serves as an alias for the type:
>
>       AliasDecl = identifier "=" Type .

As highlighted above, a **defined type** doesn't inherit any methods bound to the given type. Can we just define a new type from `Foo` inside `Foo.UnmarshalJSON()` and convert the accepter into the new type (from original type `Foo`), so that the `json.Unmarshal` will no longer call the `Foo.UnmarshalJSON()` again because the new type doesn't "inherit" the method from `Foo`.

Let's try this by modifying `Foo.UnmarshalJSON()` a bit:

```go
func (v *Foo) UnmarshalJSON(data []byte) error {
	type t Foo
	return json.Unmarshal(data, (*t)(v))
}
```

Output:

```
(main.Foo) {
 A: (string) "",
 Bar: (main.Bar) {
  B: (string) (len=30) "{\n  \"a\": \"foo\",\n  \"b\": \"bar\"\n}"
 }
}
```

We just hit the issue we met at the beginning! The reason is that although the new type (`t`) doesn't inherit the `UnmarshalJSON()` from `Foo`, but it DOES hold the `UnmarshalJSON()` promoted from `Bar` because `t` **embeds `Bar`**.

So the invocation chain is like: `json.Unmarshal(Foo)` -> `Foo.UnmarshalJSON()` -> `json.Unmarshal(t)` -> `Bar.UnmarshalJSON()`.

But one take away is that, we can use this hack to change the implementation of `Bar` so that it could correctly unmarshal the json to its strucutre, rather than simply converting into string:

```go
func (v *Bar) UnmarshalJSON(data []byte) error {
	type t Bar
	return json.Unmarshal(data, (*t)(v))
}
```

Output:

```
(main.Foo) {
 A: (string) "",
 Bar: (main.Bar) {
  B: (string) (len=3) "bar"
 }
}
```

(although `A` is still empty)

# Another Expected Broken Solution But Just Works?!

As curious as me, I'm just wondering what will happen if I embed another type into `Foo`. So I modified the code as below:

```go
type Foo struct {
	A string
	Bar
	Baz // embed a second type
}

type Baz struct {
	C string
}

func (v *Baz) UnmarshalJSON(data []byte) error {
	type t Baz
	return json.Unmarshal(data, (*t)(v))
}

func main() {
    //...
    // Add a field "c" in the input json
	if err := json.Unmarshal([]byte(`{
  "a": "foo",
  "b": "bar",
  "c": "baz"
}`), &foo); err != nil {
    //...
}
```

Output:

```
(main.Foo) {
 A: (string) (len=3) "foo",
 Bar: (main.Bar) {
  B: (string) (len=3) "bar"
 },
 Baz: (main.Baz) {
  C: (string) (len=3) "baz"
 }
}
```

What?! This time the result is just as expected?! This really surprised me.

So I seek help from the community, and even submitted an [issue](https://github.com/golang/go/issues/39175) in Go repo. The reason of this is actually mentioned in the Go Spec I quoted above:

>  A field or method f of an embedded field in a struct x is called **promoted** if **x.f is a legal selector that denotes that field or method f**

So what does **legal selector** actually means?

[*From Go Spec of Selector*](https://golang.org/ref/spec#Selectors):

> For a primary expression x that is not a package name, the selector expression: `x.f` denotes the field or method f of the value x (or sometimes *x; see below). The identifier f is called the (field or method) selector
>
> A selector f may denote a field or method f of a type T, or it may refer to a field or method f of a nested embedded field of T. The number of embedded fields traversed to reach f is called its **depth** in T. The depth of a field or method f declared in T is zero. The depth of a field or method f declared in an embedded field A in T is the depth of f in A plus one.
>
> The following rules apply to selectors:

> 1. For a value x of type T or *T where T is not a pointer or interface type, x.f denotes the field or method at the shallowest depth in T where there is such an f. If there is not **exactly one** f with shallowest depth, the selector expression is illegal.
> 1. For a value x of type I where I is an interface type, x.f denotes the actual method with name f of the dynamic value of x. If there is no method with name f in the method set of I, the selector expression is illegal.
> 1. As an exception, if the type of x is a defined pointer type and (*x).f is a valid selector expression denoting a field (but not a method), x.f is shorthand for (*x).f.
> 1. In all other cases, x.f is illegal. 
> 1. If x is of pointer type and has the value nil and x.f denotes a struct field, assigning to or evaluating x.f causes a run-time panic.
> 1. If x is of interface type and has the value nil, calling or evaluating the method x.f causes a run-time panic.

From the definition above, we can see that we are violating the 1st rule here, as the selector `t.UnmarshalJSON()` has **two** `UnmarsalJSON()` in the same depth (1), so Go doesn't allow the method promoted into method set of `t`. And `json` just fallback to its default unmarshal implementation.

The invocation chain is as: `json.Unmarshal(Foo)` -> `Foo.UnmarshalJSON()` -> `json.Unmarshal(t)` -> `json` default unmarshal logic (`t`) (where `t` is a type derived from `Foo`).

# Proposed Solution?

For this specific issue about `json` unmarshaling, I was thinking that if it make sense to add more check during determining whether to use a custom `Unmarshaler`. For example, maybe we can just avoid to use any promoted method at all, and only check for the method with depth of zero.
In this case, we can always be sure when our type (e.g. `Foo`) is unmarshaled, whether it will use our own custom `Unmarshaler` (e.g. `Foo.UnmarshalJSON`) or the json default impl, no matter what type we are embedding (e.g. `Bar`).

However, after some discussion with community, it seems current logic is a Go idiomatic way of handling nested fields. For example, if user is using a function which expecting parameter implemented `Stringehal`, it make sense that the passed in type could just leverage the embedded type's `String()`, rather than define its own.

So problem in our `json` case here is actually not the problem of Go, but the problem of how we implement `Unmarshaler`!

# Real Solution!

The correct way to implement `Unmarshaler` is that, rather than offload all the json bytes unmarshaling job into embedded field, we should unmarshal the part belongs to current type at the `UnmarshalJSON()` definition of current type.

For example, we should unmarshal the `A` part in `Foo.UnmarshalJSON()`, then we can offload the other part to `Bar.UnmarshalJSON()`. And in `Bar.UnmarshalJSON()`, it should only handle `B` part. And as there is no more embedded field in `Bar`, it should just stop there.

```go
package main

import (
	"encoding/json"
	"log"

	"github.com/davecgh/go-spew/spew"
)

type Foo struct {
	A string
	Bar
	Baz
}

type Bar struct {
	B string
}

type Baz struct {
	C string
}

func (input *Foo) UnmarshalJSON(data []byte) error {
	var m map[string]json.RawMessage

	if err := json.Unmarshal(data, &m); err != nil {
		return err
	}

	for k, v := range m {
		switch k {
		case "a":
			var a string
			if err := json.Unmarshal(v, &a); err != nil {
				return err
			}
			input.A = a
		}
	}
	delete(m, "a")

	b, err := json.Marshal(m)
	if err != nil {
		return err
	}

	var bar Bar
	if err := json.Unmarshal(b, &bar); err != nil {
		return err
	}
	input.Bar = bar

	var baz Baz
	if err := json.Unmarshal(b, &baz); err != nil {
		return err
	}
	input.Baz = baz

	return nil
}

func (input *Bar) UnmarshalJSON(data []byte) error {
	var m map[string]json.RawMessage

	if err := json.Unmarshal(data, &m); err != nil {
		return err
	}

	for k, v := range m {
		switch k {
		case "b":
			var b string
			if err := json.Unmarshal(v, &b); err != nil {
				return err
			}
			input.B = b
		}
	}
	delete(m, "b")
	return nil
}

func (input *Baz) UnmarshalJSON(data []byte) error {
	var m map[string]json.RawMessage

	if err := json.Unmarshal(data, &m); err != nil {
		return err
	}

	for k, v := range m {
		switch k {
		case "c":
			var c string
			if err := json.Unmarshal(v, &c); err != nil {
				return err
			}
			input.C = c
		}
	}
	delete(m, "c")
	return nil
}

func main() {
	var foo Foo
	if err := json.Unmarshal([]byte(`{
  "a": "foo",
  "b": "bar",
  "c": "baz"
}`), &foo); err != nil {
		log.Fatal(err)
	}
	spew.Dump(foo)
}
```

Note that if `Foo`, `Bar` and `Baz` has some conflict fields, the library will just silent the error, so you may need to ensure that will not happen (maybe by enabling `Decoder.DisallowUnknownFields`).

# Final Thought

Quote from Daniel's reply in my earlier [issue](https://github.com/golang/go/issues/39175):

> I think the general answer here is - if you don't know what methods a named type has, or if it's maintained in an external module out of your control, you want to be careful about embedding it and gaining its methods. Because it might "take over" methods you don't declare yourself and affect printing, encoding, decoding, etc.

So avoid the nested field if you can :)
