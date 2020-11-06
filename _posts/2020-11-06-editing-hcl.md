---
layout: "post"
title: "Edit HCL For Terraform"
categories: "blog"
tags: ['hcl']
published: True
comments: true
script: [post.js]
excerpted: |
    This is a peek on the hcl(v2) package set.
---

* TOC
{:toc}

# Addressing

## Traversal

> A Traversal is a description of traversing through a value through a series of operations such as attribute lookup, index lookup, etc.
>
> It is used to look up values in scopes, for example.
>
> A traversal can be absolute (its first value is a symbol name) or relative (starts from an existing value).

The definition of `Traversal` is as below:

```go
type Traversal []Traverser
```

where the `Traverser` is an interface.  This is a closed set of implementations, so the interface cannot be implemented from outside this package. These implementations include:

* `TraverseRoot`: `TraverseRoot` looks up a root name in a scope. It is used as the first step of an absolute `Traversal`, and cannot itself be traversed directly.

    In other word, if the first `Traverser` in the `Traversal` is `TraverseRoot`, then this `Traversal` isan absolute `Traversal`:

	```go
	func (t Traversal) IsRelative() bool {
		if len(t) == 0 {
			return true
		}
		if _, firstIsRoot := t[0].(TraverseRoot); firstIsRoot {
			return false
		}
		return true
	}
	```

* `TraverseAttr`: `TraverseAttr` looks up an attribute in its initial value.
* `TraverseIndex`: `TraverseIndex` applies the index operation to its initial value.
* `TraverseSplat`: `TraverseSplat` applies the splat operation to its initial value.

## TraversalExpr

On top of `Traversal` in the `hcl` package, `hclsyntax` defines following Traversal Expressions:

* `RelativeTraversalExpr`: an Expression that retrieves a value from another value using a _relative_ traversal.

	The definition of `RelativeTraversalExpr` is like below:

	```go
	type RelativeTraversalExpr struct {
		Source    Expression
		Traversal hcl.Traversal
		SrcRange  hcl.Range
	}
	```

	It represents the `Traversal`, relative to the `Source` expression.

	Example:

	```hcl
	resource "foo" "y" {
		count = 3
		name = foo.x[count.index].name
	}
	```

	Here, the `name` attribute in the ast tree is as below:

	```
	Expr (hclsyntaxt.RelativeTraversalExpr) {
		Source (hclsyntax.IndexExpr) {
			Collection (hclsyntax.ScopedTraversalExpr) {
				Traversal (hcl.Traversal) [
					"foo" (hcl.TraverseRoot)
					"x" (hcl.TraverseAttr)
				]
			}
			Key (hclsyntax.ScopedTraversalExpr) {
				Traversal (hcl.Traversal) [
					"count" (hcl.TraverseRoot)
					"index" (hcl.TraverseAttr)
				]
			}
		}
		Traversal (hcl.Traversal) [
			"name" (hcl.TraverseAttr)
		]
	}
	```

* `ScopeTraversalExpr`: an Expression that retrieves a value from the scope using a traversal.

	The scope could be the current module in context of Terraform when it refer to a resource, for example. It can also refer to some internal scope, as the `foo.x` expression in above example.

	Example:

	```hcl
	resource "foo" "y" {
		count = 3
		name = foo.x[0].name
	}
	```

	Here the `name` attribute in the ast tree is as below:

	```
	Expr (hclsyntax.ScopeTraversalExpr) {
		Traversal (hcl.Traversal) [
			"foo" (hcl.TraverseRoot)
			"x" (hcl.TraverseAttr)
			"[0]" (hcl.TraverseIndex)
			"name" (hcl.TraverseAttr)
		]
	}
	```