---
layout: "post"
title: "Go AST Tips"
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

# 1 Frequently Used Package

Go comes with a big bunch of packages to help user to parse and process Go code. Here I just list some frequently used ones with some brief description:

- [go/token](https://golang.org/pkg/go/token)

    Package token defines constants representing the lexical tokens of the Go programming language and basic operations on tokens (printing, predicates). 

    Some of the frequently used stuff includes:

    - `FileSet` (type): A FileSet represents a set of source files. It has a method function `FileSet.Position()` that will converts a `token.Pos` (each `ast.Node` contains `Pos()` method, which is a compact encoding of a source position within a file set) in the fileset into a `FileSet.Position` value, which contains detailed position info for a token.

    - `Token` consts (constant) (e.g. `ADD` for `+`): these constants are useful when given a `ast.Node` and some of the fields is of type `toekn.Token` and we can compare those fields with the token constants defined here. E.g. we can test against `ast.BinaryExpr.Op` to see which kind of operation this binary expression is.

- [go/parser](https://golang.org/pkg/go/parser/)

    Package parser implements a parser for Go source files. Input may be provided in a variety of forms; the output is an abstract syntax tree (AST) representing the Go source.

    Some of the frequently used stuff includes:

    - `ParseFile` (func): ParseFile parses the source code of a single Go source file and returns the corresponding `ast.File` node.

    Most parse functions provided accept a `mode` function, user can specify one or more mode flag to control parse behaviros. E.g. by adding `ParseComments` flag to parse the comment in source code into ast also.

- [go/ast](https://golang.org/pkg/go/ast)

    Package ast declares the types used to represent syntax trees for Go packages.

    Some of the frequently used stuff includes:

    - `Walk` / `Inspect` (func):  Both traverses an AST in depth-first order, except `Walk` has more control on traverse process (e.g. skip some parent node).

    There are tons of different types implement `ast.Node`, turn back to search for what fits you.

- [go/format](https://golang.org/pkg/go/format/)

    Package format implements standard formatting of Go source.

    Note that formatting of Go source code changes over time, so tools relying on consistent formatting should execute a specific version of the gofmt binary instead of using this package. That way, the formatting will be stable, and the tools won't need to be recompiled each time gofmt changes.

    For example, pre-submit checks that use this package directly would behave differently depending on what Go version each developer uses, causing the check to be inherently fragile.

- [go/printer](https://golang.org/pkg/go/printer/)

    Package printer implements printing of AST nodes.

- [go/types](https://golang.org/pkg/go/types)

    Package types declares the data types and implements the algorithms for type-checking of Go packages. Use `Config.Check` to invoke the type checker for a package. Alternatively, create a new type checker with `NewChecker` and invoke it incrementally by calling `Checker.Files`.

    Type-checking consists of several interdependent phases:

    - **Name Resolution** maps each identifier (`ast.Ident`) in the program to the language object (Object) it denotes. Use `Info.{Defs,Uses,Implicits}` for the results of name resolution.

    - **Constant Folding** computes the exact constant value (`constant.Value`) for every expression (`ast.Expr`) that is a compile-time constant. Use `Info.Types[expr]`.Value for the results of constant folding.

    - **Type Inference** computes the type (Type) of every expression (`ast.Expr`) and checks for compliance with the language specification. Use Info.`Types[expr].Type` for the results of type inference.

    There are just too much info here, so there even exist a [tutorial](https://github.com/golang/example/tree/master/gotypes) for this package.

    For introduction, I just add a very simple example about how information is gathered and being used later (e.g. when traversing the ast):
    
    ```go
    // import "go/types" and "go/importer"
    conf := types.Config{Importer: importer.Default()}

    // types.TypeOf() requires all three maps are populated
    info := &types.Info{
        Defs:  make(map[*ast.Ident]types.Object),
        Uses:  make(map[*ast.Ident]types.Object),
        Types: make(map[ast.Expr]types.TypeAndValue),
    }

    _, err = conf.Check("addlint", fset, files, info)
    if err != nil {
        log.Fatalln(err)
    }
    ```

    Above code will type check all the `files` (of type `[]*ast.File`) and then populate the `info` variable's map with all the information asked for (specified in `info`). 
    
    Afterwards, you can query the information of an `ast.Expr` by calling `info.TypeOf()`, which returns a `type.Type` interface.

Especially for code analysis, go provides a set of analysis packages:

- [analysis](https://godoc.org/golang.org/x/tools/go/analysis)

    The analysis package defines the interface between a modular static analysis and an analysis driver program.

# 99 Tips

## 99.1 Lossy Comment Handling During AST -> Source Code

```go
type example struct {
    foo int

    // this is a lossy comment

    bar int
}
```

Each field is of type `*ast.Field`. This struct has a `*ast.Field.Comment` field that contains the comment for that specific field.

But for the example above, to whom does it belong? Is it a part of `foo` or `bar`?

Because it’s **impossible** to determine it, these comments are called loosely comments. Now the problem occurs if you print the struct above with the `format.Node()` function. When you print it, this is what you get:

```go
type example struct {
    foo int

    bar int
}
```

The problem is that lossy comments are **part** of the `*ast.File` and hold separated from the tree. And those are only printed when you print the **whole** file. So the workaround this is to print the whole file and then cut out the specific lines we want to return.

## 99.2 Replace ast Node

By leveraging the `astutil.Apply()`, we can replace or manipulate node in a AST. However, something to notice that:

1. If you replace a node, then `astutil.Apply()` will not traverse both the new node and old node. So if you want to recursively apply the containing node of the new node, you need to explicitly call `astutil.Apply()` on the new node.
2. If you replace a node via `astutil.Apply()` during `ast.Walk()`, then `ast.Walk()` will not traverse both the new noed and the old node. If you still want to walk the new node, you need to explicitly call `ast.Walk()` on the new node. (same for `ast.Inspect()`)

## 99.3 Fill ast node parent

Refered to [gogrep](https://github.com/mvdan/gogrep/blob/master/match.go#L32).

```go
type nodeList interface {
	at(i int) ast.Node
	len() int
	slice(from, to int) nodeList
	ast.Node
}

func (m *matcher) fillParents(nodes ...ast.Node) {
	stack := make([]ast.Node, 1, 32)
	for _, node := range nodes {
		inspect(node, func(node ast.Node) bool {
			if node == nil {
				stack = stack[:len(stack)-1]
				return true
			}
			if _, ok := node.(nodeList); !ok {
				m.parents[node] = stack[len(stack)-1]
			}
			stack = append(stack, node)
			return true
		})
	}
}

func (m *matcher) matches(cmds []exprCmd, nodes []ast.Node) []ast.Node {
	m.parents = make(map[ast.Node]ast.Node)
    m.fillParents(nodes...)
    //...
}
```