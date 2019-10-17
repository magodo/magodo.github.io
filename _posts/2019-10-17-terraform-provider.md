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