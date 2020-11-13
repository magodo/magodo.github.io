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

## Addressing

### Traversal

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

### TraversalExpr

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

### Analyzing Traversal

In `hcl` package, the `Expression` interface requires a method called `Variables()`, which returns a list of **absolute Traversals** referenced in the receiving expression. However, this is not true. For example, it can not handle following cases:

```hcl
resource "foo" "y" {
	count = 3
	name = foo.x[count.index].name
}
```

The resulting `Variables()` are as below:

```
[
	Traversal [
		"foo" (hcl.TraverseRoot)
		"x" (hcl.TraverseAttr)
	],
	Traversal [
		"count" (hcl.TraverseRoot)
		"index" (hcl.TraverseAttr)
	],
]
```

Where the `name` is missed.

Similarly, in `terraform/lang` package, the `ReferencesInExpr()` inherits the same limitation since it is built on top of the `Variables()` function above. So as `RenameVariablePrefix()` in `hcl/hclwrite` package does.

> 💡 The reason that `Variables()` not work for above hcl is because the internal walking logic for `hclsyntax.RelativeTraversalExpr` only tackles with its `Source` component, but ignore the `Traversal` component at all.

## Expression

This section we are going to talk about all kinds of expressions exist in HCL, as defined in `hcl/hclsyntax` package.

There are two categories of expressions:

* template expression: for string template (including string literal)
* general expression: all other expressions except `template expression`

### General Expression

#### ScopeTraversalExpr and RelativeTraversalExpr

`ScopeTraversalExpr` is an absolute traversal, which contains a list of `Traverse`.

`RelativeTraversalExpr` is contains a `Source` expression and a relative traversal related to the `Source`, which is a list of `Traverse`.

#### LiteralValueExpr

This is simpliy the literal value expression, could be a number or a bool:

```hcl
a = 1
b = true
c = null
```

#### TupleConsExpr

This could represent a list or a set of `expression`s:

```hcl
a = [1,2,3]
```

Above is a list of `LiteralValueExpr`.

#### FunctionCallExpr

The function call expression, its argument can be a list of `expression`s.

See [this](https://www.terraform.io/docs/configuration/expressions.html#function-calls).

#### ConditionalExpr

The conditional expression as below:

```hcl
a = 1==1 ? 0 : 1
```

Its `Conditional`, `TrueResult` and `FalseResult` components could all be `expression`s.

See [this](https://www.terraform.io/docs/configuration/expressions.html#conditional-expressions).

#### IndexExpr

The expression appears in the index, like the `b` in below example:

```hcl
a = l[0].name
b = l[count.index].name
```

The ast is as below:

```
	 ...
     3  .  .  .  "a": *hclsyntax.Attribute {
     4  .  .  .  .  Name: "a"
     5  .  .  .  .  Expr: *hclsyntax.ScopeTraversalExpr {
     6  .  .  .  .  .  Traversal: hcl.Traversal (len = 3) {
     7  .  .  .  .  .  .  0: hcl.TraverseRoot {
     8  .  .  .  .  .  .  .  Name: "l"
     9  .  .  .  .  .  .  }
    10  .  .  .  .  .  .  1: hcl.TraverseIndex {
    11  .  .  .  .  .  .  .  Key: cty.Value {}
    12  .  .  .  .  .  .  }
    13  .  .  .  .  .  .  2: hcl.TraverseAttr {
    14  .  .  .  .  .  .  .  Name: "name"
    15  .  .  .  .  .  .  }
    16  .  .  .  .  .  }
    17  .  .  .  .  }
    18  .  .  .  }
    19  .  .  .  "b": *hclsyntax.Attribute {
    20  .  .  .  .  Name: "b"
    21  .  .  .  .  Expr: *hclsyntax.RelativeTraversalExpr {
    22  .  .  .  .  .  Source: *hclsyntax.IndexExpr {
    23  .  .  .  .  .  .  Collection: *hclsyntax.ScopeTraversalExpr {
    24  .  .  .  .  .  .  .  Traversal: hcl.Traversal (len = 1) {
    25  .  .  .  .  .  .  .  .  0: hcl.TraverseRoot {
    26  .  .  .  .  .  .  .  .  .  Name: "l"
    27  .  .  .  .  .  .  .  .  }
    28  .  .  .  .  .  .  .  }
    29  .  .  .  .  .  .  }
    30  .  .  .  .  .  .  Key: *hclsyntax.ScopeTraversalExpr {
    31  .  .  .  .  .  .  .  Traversal: hcl.Traversal (len = 2) {
    32  .  .  .  .  .  .  .  .  0: hcl.TraverseRoot {
    33  .  .  .  .  .  .  .  .  .  Name: "count"
    34  .  .  .  .  .  .  .  .  }
    35  .  .  .  .  .  .  .  .  1: hcl.TraverseAttr {
    36  .  .  .  .  .  .  .  .  .  Name: "index"
    37  .  .  .  .  .  .  .  .  }
    38  .  .  .  .  .  .  .  }
    39  .  .  .  .  .  .  }
    40  .  .  .  .  .  }
    41  .  .  .  .  .  Traversal: hcl.Traversal (len = 1) {
    42  .  .  .  .  .  .  0: hcl.TraverseAttr {
    43  .  .  .  .  .  .  .  Name: "name"
    44  .  .  .  .  .  .  }
    45  .  .  .  .  .  }
    46  .  .  .  .  }
    47  .  .  .  }
	...
```

Here, the `[0]` in attribute `a` is NOT a `IndexExpr`, but a `hcl.TraverseIndex`. While the `l[count.index]` in attribute `b` is a `IndexExpr`, which contains `Collection`, describing the collection subject on which will index by key, here it is `l`. Also, it contains `Key`, describing the index applies to the `Collection`.

Both `Collection` and `Key` are `expression`s.

#### ObjectConsExpr and ObjectConsKeyExpr

This describes either a map or a object. It contains a list of `ObjectConsItem`s, where each item is consisted of a `KeyExpr` and a `ValueExpr`.

The `ValueExpr` could be arbitrary `expression`.

While `KeyExpr` are strings and is always a `ObjectConsKeyExpr`, which is a special wrapper used only for `ObjectConsExpr` keys. It deals with the special case that a naked identifier in that postion must be interpreted as a literal string rather than evaluated directly.

In Terraform, the key has following valid forms:

```hcl
m = {
	k1 		= "v1"     	# one length traversal, without quotes. it will be transformed to string
	(var.a) = "v2"  	# more than one length traversal, with parenthes
	"foo" 	= "v3"		# string literal
}
```

The ast is as below:

```
	...
     7  .  .  .  .  .  .  0: hclsyntax.ObjectConsItem {
     8  .  .  .  .  .  .  .  KeyExpr: *hclsyntax.ObjectConsKeyExpr {
     9  .  .  .  .  .  .  .  .  Wrapped: *hclsyntax.ScopeTraversalExpr {
    10  .  .  .  .  .  .  .  .  .  Traversal: hcl.Traversal (len = 1) {
    11  .  .  .  .  .  .  .  .  .  .  0: hcl.TraverseRoot {
    12  .  .  .  .  .  .  .  .  .  .  .  Name: "k1"
    13  .  .  .  .  .  .  .  .  .  .  }
    14  .  .  .  .  .  .  .  .  .  }
    15  .  .  .  .  .  .  .  .  }
    16  .  .  .  .  .  .  .  }
	...
    24  .  .  .  .  .  .  }
    25  .  .  .  .  .  .  1: hclsyntax.ObjectConsItem {
    26  .  .  .  .  .  .  .  KeyExpr: *hclsyntax.ObjectConsKeyExpr {
    27  .  .  .  .  .  .  .  .  Wrapped: *hclsyntax.ScopeTraversalExpr {
    28  .  .  .  .  .  .  .  .  .  Traversal: hcl.Traversal (len = 2) {
    29  .  .  .  .  .  .  .  .  .  .  0: hcl.TraverseRoot {
    30  .  .  .  .  .  .  .  .  .  .  .  Name: "var"
    31  .  .  .  .  .  .  .  .  .  .  }
    32  .  .  .  .  .  .  .  .  .  .  1: hcl.TraverseAttr {
    33  .  .  .  .  .  .  .  .  .  .  .  Name: "a"
    34  .  .  .  .  .  .  .  .  .  .  }
    35  .  .  .  .  .  .  .  .  .  }
    36  .  .  .  .  .  .  .  .  }
    37  .  .  .  .  .  .  .  }
	...
    45  .  .  .  .  .  .  }
    46  .  .  .  .  .  .  2: hclsyntax.ObjectConsItem {
    47  .  .  .  .  .  .  .  KeyExpr: *hclsyntax.ObjectConsKeyExpr {
    48  .  .  .  .  .  .  .  .  Wrapped: *hclsyntax.TemplateExpr {
    49  .  .  .  .  .  .  .  .  .  Parts: []hclsyntax.Expression (len = 1) {
    50  .  .  .  .  .  .  .  .  .  .  0: *hclsyntax.LiteralValueExpr {
    51  .  .  .  .  .  .  .  .  .  .  .  Val: cty.Value {}
    52  .  .  .  .  .  .  .  .  .  .  }
    53  .  .  .  .  .  .  .  .  .  }
    54  .  .  .  .  .  .  .  .  }
    55  .  .  .  .  .  .  .  }
	...
    63  .  .  .  .  .  .  }
	...
```

From above, the `Wrapped` component of a `ObjectConsKeyExpr` can be either `ScopeTraversalExpr` or `TemplateExpr`.

See the last category of [this section](https://www.terraform.io/docs/configuration/expressions.html#literal-expressions).

See [the code](https://github.com/magodo/terraform/blob/abec0acf40d8e31ac612a244cf3886fb3bcce0bb/vendor/github.com/hashicorp/hcl2/hcl/hclsyntax/expression.go#L837-L837) where it tackle with more than one lenght traversal.

#### ForExpr

TBD

#### SplatExpr

TBD

#### AnonSymbolExpr

TBD
