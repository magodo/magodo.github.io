---
layout: "post"
title: "Go JSON Marshal/Unmarshal"
categories: "blog"
tags: ["go"]
published: True
comments: true
script: [post.js]
excerpted: |
---

- TOC
  {:toc}

This post talks about some black corners of Go's `json` package, in turns of its marshal/unmarshal process.

## Kinds of Marshal/UnMarshal Cases

Marshal encodes the Go type into json byte representation, and unmarshal do the inverse thing. Let's talk about different cases regarding marshal/unmarshal, for different kinds of struct fields.

### Primary Type

Given `foo.I` is type of `int`.

For marshall:

| Go type and value | json                         |
| ----------------- | ---------------------------- |
| foo.I = 1(int)    | `{i: 1}`                     |
| foo.I = 0(int)    | `{i: 0}` / NA (if omitempty) |

For unmarshall:

| json          | Go type and value |
| ------------- | ----------------- |
| `{i: 1}`      | foo.I = 1(int)    |
| `{i: 0}` / NA | foo.I = 0(int)    |

### Pointer Type

Given `foo.I` is type of `*int`.

For marshall:

| Go type and value | json                            |
| ----------------- | ------------------------------- |
| foo.I = intPtr(1) | `{i: 1}`                        |
| foo.I = intPtr(0) | `{i: 0}`                        |
| foo.I = nil       | `{i: null}` / NA (if omitempty) |

For unmarshall:

| json             | Go type and value |
| ---------------- | ----------------- |
| `{i: 1}`         | foo.I = intPtr(1) |
| `{i: 0}`         | foo.I = intPtr(0) |
| `{i: null}` / NA | foo.I = nil       |

### Slice Type

Given `foo.L` is type of `[]int`.

For marshal:

| Go type and value | json                            |
| ----------------- | ------------------------------- |
| foo.L = nil       | `{l: null}` / NA (if omitempty) |
| foo.L = []int{}   | `{l: []}` / NA (if omitempty)   |
| foo.L = []int{1}  | `{l: [1]}`                      |

For unmarshal:

| json             | Go type and value |
| ---------------- | ----------------- |
| `{l: null}` / NA | foo.L = nil       |
| `{l: []}`        | foo.L = []int{}   |
| `{l: [1]}`       | foo.L = []int{1}  |

## Problem?

During marshalling, people wants to be able to convert some value to following different states in the json representation:

- normal value (e.g. `1`, `[]int{1,2,3}`)
- zero value (e.g. `0`, `[]int{}`)
- json `null` (e.g. `{x : null}`)
- absent key (e.g. `{}`)

While, during unmarshalling, people only cares about the decoded Go value, depending on whether it is a pointer, it can be catagorized into:

- normal value
- zero value
- (pointer only) `nil`

Apparently, even if pointer is used, we still can't get a 1:1 mapping above. Therefore, we need to create a new state in the Go type.

## Solution

In Azure Go SDK Track 2, this is solved by introducing a sentinel value: [`NullValue`](https://github.com/Azure/azure-sdk-for-go/blob/master/sdk/azcore/core.go#L151), which holds a fixed pointer for each type, to represent that type's null value. Then we can get following mappings:

```
  JSON                      Go

normal value    --      normal value
zero value      --      zero value
null            --      NullValue
absent          --      nil (omitempty)
```

The precondition is that each type is modeled as pointer (including primary type, slice, map, etc.), and `omitempty` is enabled.

Apparently, this doesn't come free. We will need to implement the custom marshaller and unmarshaller to handle the `NullValue`. Also, note that if the structure contains/is contained as anonymous field, then the custom marshaller/unmarshaller is [infectious](https://github.com/golang/go/issues/39175), which means we will need to implement all the types along the dependency chain.
