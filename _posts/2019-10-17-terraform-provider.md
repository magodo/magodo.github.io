---
layout: "post"
title: "Terraform Provider Tips"
categories: "blog"
tags: ['terraform-provider']
published: True
comments: true
script: [post.js]
excerpted: |
    关于Terraform及其Provider编写的概念和Tips...
---

* TOC
{:toc}

# 1. Terraform language fomat

[官网](https://www.terraform.io/docs/configuration/index.html#arguments-blocks-and-expressions)中对于terraform的语言格式描述如下：

```
<BLOCK TYPE> "<BLOCK LABEL>" "<BLOCK LABEL>" {
  # Block body
  <IDENTIFIER> = <EXPRESSION> # Argument
}
```

在 [官网的另一个页面](https://www.terraform.io/docs/configuration/syntax.html)，有更为详细的语法说明。其中需要注意的是 `block body` 中的内容除了 `argument`（ [某些地方](https://www.terraform.io/docs/extend/terraform-0.12-compatibility.html#configuration-syntax-changes) 也称为 `attribute` ）以外，也允许是内嵌的 `block`。

因此，上面这个spec应该更新为：

```
<BLOCK TYPE> "<BLOCK LABEL>" "<BLOCK LABEL>" {
  # Block body

  <IDENTIFIER> = <EXPRESSION> # Argument

  OR

  <IDENTIFIER> {              # Nested Block
      # Block body
  }
}
```

与之前不同的是，我在 `block body` 中加入了 `Nested Block`。需要注意的是这里 `argument` 的syntax（有 `=` 号） 和 `block` 的syntax（没有 `=` ）不同。这个限制在tf 0.12 以后是[强制](https://www.terraform.io/docs/extend/terraform-0.12-compatibility.html#configuration-syntax-changes)的。

另外， `argument` 除了Provider定义的以外，不同的 `block type` 还会有一些 [`meta argument`](https://www.terraform.io/docs/configuration/resources.html#meta-arguments).

建议，为了区分这两个 `block body` 中仅有的两类元素，遵循官方以及源码中的命名，称呼它们为：`attribute`/`argument` 和 `block`.

# 99 TIP

## 99.1 block body nested map

在 [官网](https://www.terraform.io/docs/extend/writing-custom-providers.html#implementing-a-more-complex-read)有提到：

> Due to the limitation of [tf-11115](https://github.com/hashicorp/terraform/issues/11115) it is not possible to nest maps. So the workaround is to let only the innermost data structure be of the type `TypeMap`

这里的workaround在tf 0.12版本以后就被修复了，具体可以看上面那个issue。同时，有另外一个与之关联的[issue](https://github.com/hashicorp/terraform-website/issues/908)要求把上面这个章节删除。So...如果你打不开上面那个官网的章节的话就代表已经被删除了，如果你依然可以访问，请无视 （我在这里看了好一会儿 😶）