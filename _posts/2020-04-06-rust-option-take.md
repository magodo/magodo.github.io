---
layout: "post"
title: "Rust `Option` take"
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

## Problematic Code

```rust
struct Foo {
    bar: Bar,
}

impl Foo {
    fn foo(&self) {
        self.bar.bar();
    }
}

struct Bar {}

impl Bar {
    fn bar(self) {}
}

fn main() {
    let foo = Foo { bar: Bar {} };
    foo.foo();
}
```

**Output**

```bash
💤 via 𝗥 v1.42.0 cargo build
   Compiling foo v0.1.0 (/tmp/foo)
error[E0507]: cannot move out of `self.bar` which is behind a shared reference
 --> src/main.rs:7:9
  |
7 |         self.bar.bar();
  |         ^^^^^^^^ move occurs because `self.bar` has type `Bar`, which does not implement the `Copy` trait

error: aborting due to previous error

For more information about this error, try `rustc --explain E0507`.
error: could not compile `foo`.

To learn more, run the command again with --verbose.

```

## Fix by Wrapping With `Option<T>`

The problem here is because `Bar.bar` take ownership of `self` (rather than reference as `&self`). In this case, as we are holding the reference of `bar` (i.e. `foo.bar`) and because `Bar` doesn't implement `Copy` trait, so we can not transfer the ownership out.

One fix is as below:

```rust
struct Foo {
    bar: Option<Bar>,
}

impl Foo {
    fn foo(&mut self) {
        if let Some(b) = self.bar.take() {
            b.bar();
        }
    }
}

struct Bar {}

impl Bar {
    fn bar(self) {}
}

fn main() {
    let mut foo = Foo { bar: Some(Bar {}) };
    foo.foo();
}
```

There are following changes:

1. Change type of `Foo.bar`: `Bar` -> `Option<Bar>` and change the impl of `Foo.foo`, where we use `take` method of `Option<T>` which take the value out of the `Option` and leaving a `None` in its place (this actually takes ownership of the value out).
1. As `bar` is a field in `Foo`, we need also change `Foo.foo()`'s signature from `foo(&self)` to `foo(&mut self)`.

But one thing to point out is that as we are using `take()`, which will take the value out of `Option` and leave it with `None`, after we calling `foo.foo()`, then `foo.bar` is a `None`, if we call `foo.foo()` again, it will not match the `if let` pattern again.


## Other Solutions

By referring to `rustc --explain E0507`, there are more possible ways to fix, e.g. by implementing the `Copy` trait:

```rust

//...

#[derive(Clone, Copy)]
struct Bar {}

//...
```
