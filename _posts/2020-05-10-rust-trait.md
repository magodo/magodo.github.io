---
layout: "post"
title: "Rust - Trait"
categories: "blog"
tags: ['rust']
published: True
comments: true
script: [post.js]
excerpted: |
    A short description...
---

* TOC
{:toc}

> A trait tells the Rust compiler about functionality a particular type has and can share with other types. We can use traits to define shared behavior in an abstract way.

So `trait` is similar to a feature often called interfaces in other languages, although with some differences.

On the other hand:

> We can use trait bounds to specify that a generic can be any type that has certain behavior.

We are not going to tell every detail of `trait`, you can read following sections in Rust Book for them:

- [10-2 "Traits: Defining Shared Behavior"](https://doc.rust-lang.org/book/ch10-02-traits.html)
- [17-2 "Using Trait Objects That Allow for Values of Different Types"](https://doc.rust-lang.org/book/ch17-02-trait-objects.html)
- [19-3 "Advanced Trait"](https://doc.rust-lang.org/book/ch19-03-advanced-traits.html)

Instead, I'm trying to pick and illustrate some concepts that are hard to understand at the beginning, with some examples.

# Trait With Associated Type VS Trait With Generic Type

`trait` with associated type:

```rust
trait IFoo {
    type Ret;

    fn foo(&self) -> Self::Ret;
}
```

This defines only ONE `trait`, the implementor has to specify the exact type of `Ret` during `impl`. Each implementor can only implement this `trait` for once.

E.g.

```rust
struct Foo1 {}

impl IFoo for Foo2 {
    type Ret = i32;
    fn foo(&self) -> Self::Ret {
        1
    }
}

struct Foo2 {}

impl IFoo for Foo2 {
    type Ret = string;
    fn foo(&self) -> Self::Ret {
        "a"
    }
}
```

![trait-associated](/assets/img/rust-trait/trait-associated.svg)

`trait` with generic type:

```rust
trait IFoo<T> {
    fn foo(&self) -> T;
}

```

Since the `trait` is takes generic type, it means there could be more than one "concrete" traits that a type could implement.

E.g.

```rust
struct Foo {}

impl IFoo<i32> for Foo {
    fn foo(&self) -> i32 {
        1
    }
}

impl IFoo<string> for Foo {
    fn foo(&self) -> string {
        "a"
    }
}
```

`Foo` can implement both `IFoo<i32>` `trait` and `IFoo<string>` `trait`. When using the instance of `Foo`, if we simply invoke `foo`, then compiler can not infer to a unique type and will raise an error. To fix that, we have to explicitly annotate the type:

```rust
fn main() {
    let foo = Foo {};
    let _: string = foo.foo();
    //let _ = foo.foo(); // Build Error
}
```

![trait-generic](/assets/img/rust-trait/trait-generic.svg)

**NOTE** they can exist at the same time