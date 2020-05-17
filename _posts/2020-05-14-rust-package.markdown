---
layout: "post"
title: "Rust - Package"
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

![trait-associated](/assets/img/rust-package/package.svg)

The terms:

- **crate**: A crate is a binary or library. 
- **crate root**: The crate root is a source file that the Rust compiler starts from and makes up the root module (named `crate`) of your crate `module tree`.
- **package**: A package is one or more crates that provide a set of functionality. A package contains a Cargo.toml file that describes how to build those crates.

    A package could be one of the following cases:

    - One or more binary crates
    - One library crate with zero or more binary crates

    Because:
    > A package must contain zero or one library crates, and no more. It can contain as many binary crates as you’d like, but it must contain at least one crate (either library or binary)

- **module**: A module is a piece of code wrapped by `mod` keyword (potentially prefixed `pub`). Modules let us organize code within a crate into groups for readability and easy reuse. Modules also control the privacy of items, which is whether an item can be used by outside code (public) or is an internal implementation detail and not available for outside use (private).

- **pub**: `pub` makes a `module`, `struct`, member of `struct`, `fn`, `enum`, .etc be available to be used by outside code (public).

    **NOTE**: By default, when the `enum` is marked to be `pub`, all its variants are `pub` by default.

    `pub` can be used together with `use` (e.g. `pub use crate::Foo;`) to **re-exporting** some items from current scope. This technique is always used inside *src/lib.rs* library crate root to export the items defined in other modules in the same directory.

    E.g.

    *src/lib.rs*:

    ```rust
    mod foo;
    pub use foo::Foo;
    ```

    *src/foo.rs*:

    ```rust
    struct Foo {
        //...
    }
    ```
