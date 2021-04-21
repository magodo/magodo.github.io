---
layout: post
title: "Go vs Rust: String"
categories: blog
tags: ["program", "unicode", "go", "rust"]
published: true
comments: true
excerpted: |
  Discussing about the String between Go and Rust
script: [post.js]
---

{: toc}

Both Go and Rust has built-in type `string`/`String`, which are both UTF8 encoded. Also, both are implemented as slice of bytes.

In the official document of [Go](https://blog.golang.org/strings) and [Rust](https://doc.rust-lang.org/stable/book/ch08-02-strings.html#storing-utf-8-encoded-text-with-strings), they talks about the difference between byte, Unicode Scalar Value (Rust) / Unicode Code Point (Go) (Also see [the difference between Scalar Value and Code Point](https://stackoverflow.com/questions/48465265/what-is-the-difference-between-unicode-code-points-and-unicode-scalars)), and grapheme cluster (the so-called character).

Next, let's see the language behavior for each case.

## Index and Loop

Rust doesn't allow users to directly index or loop a `String`. Due to Rust's propensity for exposing possible errors, it asks users to be more specific about **what** to index/loop. The standard library provide `chars()` (Scalar Value) and `bytes()` (bytes):

```rust
for c in "नमस्ते".chars() {
    println!("{}", c);
}

// Output:
// न
// म
// स
// ्
// त
// े

for b in "नमस्ते".bytes() {
    println!("{}", b);
}

// Output:
// 224
// 164
// ...
// 165
// 135
```

In contrast, Go always allows users to index a `string`, it is actually indexing the byte slice:

```go
s := "नमस्ते"
fmt.Println(s[0])

// Output:
// 224
```

In regard of looping, Go provides two ways. The first way is to loop by indexing, this apparently is iterating the byte slice. A `for range` loop, by contrast, decodes one UTF-8-encoded `rune` (alias to `i32`, a.k.a Code Point) on each iteration:

```go
for i := 0; i < len(s); i++ {
    fmt.Println(s[i])
}
// Output:
// 224
// 164
// ...
// 165
// 135

for idx, p := range s {
    fmt.Printf("%s (byte offset: %d)\n", string(p), idx)
}
// Output:
// न (byte offset: 0)
// म (byte offset: 3)
// स (byte offset: 6)
//  ्(byte offset: 9)
// त (byte offset: 12)
//  े(byte offset: 15)
```

Note that there is a type conversion in the `for range` loop to convert the `rune` into `string`, otherwise it will simply output the Code Point value - rather than its string representation. Also note that, the index of each rune increments by the bytes occupied by the Code Point.

## Slice

In Rust, it allows users to slice a `String` into a string slice (i.e. `&str`). However, this is a dangerous operation since it might panic the program if the slice boundary is not a valid `char` (Scalar Value) boundary.

```rust
let s = "नमस्ते";
&s[..2];
// &s[..3] // works
```

Will panic:

```bash
thread 'main' panicked at 'byte index 2 is not a char boundary; it is inside 'न' (bytes 0..3) of `नमस्ते`', src/main.rs:5:6
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

In Go, it is less strict and allows users to index into the middle of a Code Point. Meanwhile, it provides the `unicode/utf8` to allow users to (e.g.) validate a rune.

## Loop Per Grapheme Cluster (i.e. Character)

There is no builtin support for this in Rust, whilst there is a crate called [unicode-segmentation](https://crates.io/crates/unicode-segmentation) that supports this:

```rust
for g in "नमस्ते्".graphemes(true) {
    println!("{}", g);
}

// output:
// न
// म
// स्
// ते्
```

In Go, there is also no official support, but there are several community packages available: github.com/rivo/uniseg, github.com/blevesearch/segment.

## Normalization

For Rust, see: https://docs.rs/unicode-normalization; For Go, see: https://blog.golang.org/normalization.
